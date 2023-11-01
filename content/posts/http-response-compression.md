+++
title = 'HTTP response compression'
date = 2023-10-31T21:59:06+02:00
draft = false
+++
Недавно пытался оптимизировать время ответа от веб сервиса. Один из эндпоинтов отдавал данные за 10 секунд, [что слишком долго](https://www.hobo-web.co.uk/your-website-design-should-load-in-4-seconds/).   
Профилирование кода помогло понять какой шаг обработки запроса тратит большую часть времени.  
Наметился список явных проблем:

- Некоторые данные запрашиваются по несколько раз. Вынес получение данных (где возможно) в этап подготовки выполнения запроса.
- Местами обработку данных удалось сделать параллельной. В паре мест даже пришлось, наоборот, сделать последовательную обработку для ускорения.
- В каких-то местах обрабатывались заведомо неподходящие данные. Добавил пару условий в циклы и фильтров для данных.  

После этих мероприятий, запрос стал выполняться за 6 секунд. Прогресс есть, но можно лучше. Подумалось, что придется двигаться глубже, оптимизировать агрессивнее.  

Как же повезло, что в postman на глаза попался тайминг запроса. Время ответа от сервера (time to first byte/start transfer) было порядка 400 мс, а вот остальное время уходило на загрузку ответа размером 1.6 Мб.  
Подобные данные можно собрать с помощью curl ([tutorial](https://stackoverflow.com/questions/18215389/how-do-i-measure-request-and-response-times-at-once-using-curl)).
Создаем файл `curl-format.txt` с таким содержимым:  

```console
     time_namelookup:  %{time_namelookup}s\n
        time_connect:  %{time_connect}s\n
     time_appconnect:  %{time_appconnect}s\n
    time_pretransfer:  %{time_pretransfer}s\n
       time_redirect:  %{time_redirect}s\n
  time_starttransfer:  %{time_starttransfer}s\n
                     ----------\n
          time_total:  %{time_total}s\n
       size_download:  %{size_download}b\n
```

Затем, выполняем команду:  

```console
 curl -w "@curl-format.txt" -o /dev/null -i https://your.site.name/path/to/endpoint
```

после чего получим что-то подобное.  

```console
     time_namelookup:  0.098889s
        time_connect:  0.380771s
     time_appconnect:  0.768659s
    time_pretransfer:  0.769072s
       time_redirect:  0.000000s
  time_starttransfer:  0.769082s
                     ----------
          time_total:  12.259356s
       size_download:  1848536b
```

Все время между `time_starttransfer` и `time_total` тратится на загрузку ответа.  
И тут в памяти всплыла возможность сжатия HTTP ответов. Так как сервис написан на Go, то ниже будет сниппет для сжатия ответов на го.  
Сначала вариант в лоб.  

```go
package main

import (
    "bytes"
    "compress/gzip"
    "io"
    "log"
    "net/http"
    "strings"
)

func main() {
    log.Printf("Service started!")
    http.ListenAndServe("localhost:8080", http.HandlerFunc(CompressingHandler))
}

func CompressingHandler(w http.ResponseWriter, req *http.Request) {
    body, err := io.ReadAll(req.Body)
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        return
    }
    
    response := body

    if strings.Contains(req.Header.Get("Accept-Encoding"), "gzip") {
        compressedBuffer := bytes.Buffer{}
        compressWriter := gzip.NewWriter(&compressedBuffer)

        _, err := compressWriter.Write(body)
        if err != nil {
            w.WriteHeader(http.StatusInternalServerError)
            return
        }
        // Don't forget to close gzipped writer.
        // Compressed data may be corrupted if that won't be closed.
        compressWriter.Close()

        response = compressedBuffer.Bytes()
        w.Header().Set("Content-Encoding", "gzip")
    }

    w.Header().Set("Content-Type", req.Header.Get("Content-Type"))
    _, err = w.Write(response)
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        return
    }
}
```

В таком варианте напрямую в обработчик добавлен код для сжатия ответа. При этом важно обратить внимание на заголовок `Accept-Encoding`. Если в нем содержится название нужного алгоритма, то только в этом случае можно применять сжатие.  
Такой код подойдет если нужно точечно сделать сжатие ответов только для одного обработчика. На самом деле на таком варианте я и остановился потому что мне это нужно было ровно в одном месте.  
Конечно же всегда можно сделать интереснее. Попробуем сделать middleware для сжатия.  

```go
package main

import (
    "compress/gzip"
    "io"
    "log"
    "net/http"
    "strings"
)

func main() {
    log.Printf("Service started!")
    http.ListenAndServe("localhost:8080", CompressingMiddleware(http.HandlerFunc(SimpleHandler)))
}

func SimpleHandler(w http.ResponseWriter, req *http.Request) {
    body, err := io.ReadAll(req.Body)
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", req.Header.Get("Content-Type"))
    _, err = w.Write(body)
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        return
    }
}

// This struct will act as a http.ResponseWriter BUT
// with mangled .Write method.
type compressResponseWriter struct {
    io.Writer
    http.ResponseWriter
}

// We mangle this method to redirect writing to the .Writer embedded struct.
func (crw compressResponseWriter) Write(b []byte) (int, error) {
    return crw.Writer.Write(b)
}

func CompressingMiddleware(h http.Handler) http.Handler {
    return http.HandlerFunc(
        func(w http.ResponseWriter, req *http.Request) {
            // Compress only if sender accepts this encoding.
            if strings.Contains(req.Header.Get("Accept-Encoding"), "gzip") {
                // Create gzip writer.
                // Everything written to that writer will be gzipped automatically and
                // written to underlying writer (http.ResponseWriter in that case).
                gzipWriter := gzip.NewWriter(w)
                w.Header().Set("Content-Encoding", "gzip")

                // Call next handler in the chain
                h.ServeHTTP(
                    &compressResponseWriter{
                        Writer:         gzipWriter,
                        ResponseWriter: w,
                    },
                    req,
                )
                // Don't forget to close gzipped writer.
                // Compressed data may be corrupted if that won't be closed.
                gzipWriter.Close()
                return
            }

            // Call next hanler if encoding is not supported.
            h.ServeHTTP(w, req)
        },
    )
}
```

Кода получилось больше, однако так его можно просто добавить в несколько эндпоинтов без копипасты.  
Время потестировать.  
Оба варианта работают одинаково, поэтому приведу тут примеры запросов и ответов.  

```console
➤ xh localhost:8080/ lol=kek accept-encoding:"gzip"
HTTP/1.1 200 OK
Content-Encoding: gzip
Content-Length: 37
Content-Type: application/json
Date: Tue, 31 Oct 2023 19:05:39 GMT

{
    "lol": "kek"
}
```

Так как curl не умеет автоматически разархивировать ответы, то придется перенаправить вывод в gunzip.  

```console
➤ curl http://localhost:8080/ -H 'accept-encoding: gzip' -H 'content-type: application/json' -H 'accept: application/json, */*;q=0.5' -d '{"lol":"kek"}'
Warning: Binary output can mess up your terminal. Use "--output -" to tell
Warning: curl to output it to your terminal anyway, or consider "--output
Warning: <FILE>" to save to a file.
```

```console
➤ curl http://localhost:8080/ -H 'accept-encoding: gzip' -H 'content-type: application/json' -H 'accept: application/json, */*;q=0.5' -d '{"lol":"kek"}' -l | gunzip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    50  100    37  100    13   4362   1532 --:--:-- --:--:-- --:--:-- 50000
{"lol":"kek"}
```

Если не указывать `Accept-Encoding`, то все будет работать как обычно.  

```console
➤ xh localhost:8080/ lol=kek accept-encoding:
HTTP/1.1 200 OK
Content-Length: 13
Content-Type: application/json
Date: Tue, 31 Oct 2023 19:16:51 GMT

{
    "lol": "kek"
}
```

Для профессионалов curl.

```console
➤ curl http://localhost:8080/ -H 'content-type: application/json' -H 'accept: application/json, */*;q=0.5' -d '{"lol":"kek"}' -i
HTTP/1.1 200 OK
Content-Type: application/json
Date: Tue, 31 Oct 2023 19:27:48 GMT
Content-Length: 13

{"lol":"kek"}
```

На таком игрушечном примере правда получилось, что со сжатием данных больше чем без сжатия. В реальных условиях начиная с какого-то объема сжатие начнет работать как сжатие.  
Вот что было у меня до приседаний со сжатием.

```console
     time_namelookup:  0.098889s
        time_connect:  0.380771s
     time_appconnect:  0.768659s
    time_pretransfer:  0.769072s
       time_redirect:  0.000000s
  time_starttransfer:  0.769082s
                     ----------
          time_total:  12.259356s
       size_download:  1848536b
```

И вот что стало после.

```console
     time_namelookup:  0.088122s
        time_connect:  0.391411s
     time_appconnect:  0.694900s
    time_pretransfer:  0.695495s
       time_redirect:  0.000000s
  time_starttransfer:  0.695510s
                     ----------
          time_total:  1.989997s
       size_download:  237117b
```

Эндпоинт тот же. Деплоймент тот же. Единственное отличие между запросами это что во втором есть заголовок `Accept-Encoding: gzip`.  
Размер данных сжался с 1848536 байт (1.76 MiB) до 237117 байт (231.6 KiB), соответственно, и время на загрузку сильно сократилось.  
Сжимайте ответы...

```text
                                                                      
                              . ...........                           
               ...   ....     . ............                          
              .....-==+:....  .....:+++*#*=:....                      
            .....-+====+=..   ....:+====-=*#-...                      
         .......========+.........-+-=**===*-......                   
          .....+==+*#+==+:........+==****+-+*......                   
           ...:*==****==+=........++-+###+==%:.....                   
          ....-+=-+**+=+*##*##****##****+==++......                   
          .....-+=-=+++**************#**+**+.......                   
           .....:*#%@%@@@@%##**####**+=++***:......                   
         ......-++*@@@@#++%@@@@@*+*@@@@@%+-==*:......                 
         ...--#*==+#@@%+*=#@#%@#+*=#@@@%+=====+*+-...  ...    ..      
         ..-#*+====+#@%+=*%#+*%%++=%@@*=======*#=..............       
         ...:*+======+######***#%#%%*=======++=.......-=+++:....      
          ....-*+=+===*+*@@@@@%++*========++*:....=+++===+*%:...      
          ......+#+==+*=*@@@@@#=-+*=====*%%+...:**+******+=#=...      
           .......:+*+*-==**+===-+*==+*###*=:.:#*********+=#...       
               .....+%%#+#@@@%#===#**++#*#***%#*********++#-....      
               ....:*%++**#@@@@%+=*+=-=#*####*##******++*+...         
             ....:+##*===*#%####++*=====****##**%#+==+**:....         
             ...:#**+====+#%##%*=*========*#*%#**###=........         
          ......***+===+++*#%#%+*+====+====**##***##:.....            
          .....:#**===**+*#++++*+==+*###===+###+#***#-....            
          .....##*+==+%+===**#*+*#%###%##==+#*#==+#*##=......         
          ....:%#*==+##*====##-..:.....:+#+=##%*==+**#%+.....         
          ....:%**==%=:#====#:...........*+=##%*#*++*#*#:....         
          ....:%*+=+%:.:*+==#............*++###:.:#*+*##*....         
          ....-%*==*#....=#=+*...........++=*##:...+#==**....         
          ....=%+==*=.....-#++#=.........*+=+#%:...-#+=#:....         
          ....*+===#-......%=-=*%:.......*===+%:...:#==#.....         
          ....#+===#:.....:#===*-.......:#===+#....**==*.....         
          ...-*===+*......:===#:........-#===+#...:*====:....         
     ........=++==++..:..:===*:.........-#===+*:...:=====::.......    
     .......-*====+*====+==+*...........:#=====:.....:=======-.....   
  ........-======+*++*====-:.... ........*=====++=:....:-==++==....   
  ......:=++=+=-=:.............    ......=*=+==++++:...............   
     ....:===+-:...............    .......=##**+++:.....              
...  ... ...                            ....     ....                 
```

Попробуйте [xh](https://github.com/ducaale/xh) вместо curl!!!  
