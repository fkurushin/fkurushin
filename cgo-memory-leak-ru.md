## Описание:

Есть код на го, использующий [faiss](https://github.com/DataIntelligenceCrew/go-faiss), то есть под капотом c++. Мне необходимо по крону совершать определенные действия с `faiss index`: скачать и обработать данные, обучить на этих данных index, затем обновить файлы новым индексом. График потребления памяти выглядит следующим образом:  
![alt text](https://github.com/fkurushin/fkurushin/blob/master/photo_2023-06-07%2014.53.12.jpeg)

_Изучение pprof и gc:_ все освобождается, но память не возвращается в ОС. 

## Обзор топиков на github и stackoverflow:


[Топик](https://github.com/facebookresearch/faiss/wiki/FAQ#why-does-the-ram-usage-not-go-down-when-i-delete-an-index)  написанный разработчиком faiss содержит в себе ценную информацию о работе деструктора С `free` более подробно можно прочитать [здесь](https://stackoverflow.com/questions/15139436/why-free-doesnt-really-frees-memory/15139468#15139468) и [здесь](https://stackoverflow.com/questions/63933234/memory-leak-when-calling-c-malloc-c-free-in-goroutines) (все эти статьи также ссылаются друг на друга)

[Проблема очень схожая с моей](https://github.com/golang/go/issues/53440)

_Вывод:_ [Крон](https://github.com/robfig/cron) запсукает программный код в отдельной горутине внутри которой исполняется с++ код, используя cgo. Деструкторы освобождают память и отдают ее в процесс в котором выполняестя прогамма. У горутин запсукаемых по крону должна быть общая память и ресурсы (из определения горутин), но получается так, что эта память блокируется
  ```
  free не обязательно возвращает память операционной системе. Чаще всего он просто возвращает эту область памяти в список свободных блоков. 
  Эти свободные блоки могут быть повторно использованы для следующих вызовов malloc.
  ```

_Гипотеза:_ Проблема заключается в том, что с-шный код запсукаемый в отдельных горутинах блокирует память и не делится ей с остальными горутинами. Нужно сделать так, чтобы `free` сразу отдавал память в ОС.

## Решение, которое мне подошло:

  - использование `index.Delete()` - не решило проблему до конца, но стало выглядеть лучше:


![alt text](https://github.com/fkurushin/fkurushin/blob/master/photo_2023-06-07%2014.20.29.jpeg)

  - Использование jemalloс: Cделал fork [go-faiss](https://github.com/DataIntelligenceCrew/go-faiss), добавил туда `#cgo LDFLAGS: -ljemalloc`, создал тег, скачал, настроил переменные окружения: 
```
# command-line-arguments
/opt/homebrew/Cellar/go/1.19.5/libexec/pkg/tool/darwin_arm64/link: running /opt/homebrew/opt/llvm/bin/clang failed: exit status 1
Undefined symbols for architecture arm64:
  "_faiss_IDSelectorBatch_new", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_IDSelectorBatch_new in 000011.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_IDSelectorBatch_new)
  "_faiss_IDSelectorRange_new", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_IDSelectorRange_new in 000011.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_IDSelectorRange_new)
  "_faiss_IDSelector_free", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_IDSelector_free in 000011.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_IDSelector_free)
  "_faiss_IndexFlat_cast", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_IndexFlat_cast in 000009.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_IndexFlat_cast)
  "_faiss_IndexFlat_new_with", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_IndexFlat_new_with in 000009.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_IndexFlat_new_with)
  "_faiss_IndexFlat_xb", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_IndexFlat_xb in 000009.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_IndexFlat_xb)
  "_faiss_Index_add", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_Index_add in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_Index_add_with_ids, __cgo_907c18e046ea_Cfunc_faiss_Index_add )
  "_faiss_Index_add_with_ids", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_Index_add_with_ids in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_Index_add_with_ids)
  "_faiss_Index_d", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_Index_d in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_Index_d)
  "_faiss_Index_free", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_Index_free in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_Index_free)
  "_faiss_Index_is_trained", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_Index_is_trained in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_Index_is_trained)
  "_faiss_Index_metric_type", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_Index_metric_type in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_Index_metric_type)
  "_faiss_Index_ntotal", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_Index_ntotal in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_Index_ntotal)
  "_faiss_Index_range_search", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_Index_range_search in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_Index_range_search)
  "_faiss_Index_remove_ids", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_Index_remove_ids in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_Index_remove_ids)
  "_faiss_Index_reset", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_Index_reset in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_Index_reset)
  "_faiss_Index_search", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_Index_search in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_Index_search)
  "_faiss_Index_train", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_Index_train in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_Index_train)
  "_faiss_ParameterSpace_free", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_ParameterSpace_free in 000006.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_ParameterSpace_free)
  "_faiss_ParameterSpace_new", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_ParameterSpace_new in 000006.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_ParameterSpace_new)
  "_faiss_ParameterSpace_set_index_parameter", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_ParameterSpace_set_index_parameter in 000006.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_ParameterSpace_set_index_parameter)
  "_faiss_RangeSearchResult_free", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_RangeSearchResult_free in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_RangeSearchResult_free)
  "_faiss_RangeSearchResult_labels", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_RangeSearchResult_labels in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_RangeSearchResult_labels)
  "_faiss_RangeSearchResult_lims", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_RangeSearchResult_lims in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_RangeSearchResult_lims)
  "_faiss_RangeSearchResult_new", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_RangeSearchResult_new in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_RangeSearchResult_new)
  "_faiss_RangeSearchResult_nq", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_RangeSearchResult_nq in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_RangeSearchResult_nq)
  "_faiss_get_last_error", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_get_last_error in 000007.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_get_last_error)
  "_faiss_index_factory", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_index_factory in 000008.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_index_factory)
  "_faiss_read_index_fname", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_read_index_fname in 000010.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_read_index_fname)
  "_faiss_write_index_fname", referenced from:
      __cgo_907c18e046ea_Cfunc_faiss_write_index_fname in 000010.o
     (maybe you meant: __cgo_907c18e046ea_Cfunc_faiss_write_index_fname)
ld: symbol(s) not found for architecture arm64
clang-16: error: linker command failed with exit code 1 (use -v to see invocation)
```

## Обзор статей в интернете по этой проблеме:

В настоящее время я читаю много статей в Интернете о проблеме, с которой я столкнулся в myjob: утечка памяти в приложении gaoling, вот несколько примеров:
1. https://www.freecodecamp.org/news/how-i-investigated-memory-leaks-in-go-using-pprof-on-a-large-codebase-4bec4325e192/
2. https://www.nylas.com/blog/finding-memory-leak-in-go-service-dev/
3. https://dev.to/googlecloud/finding-and-fixing-memory-leaks-in-go-1k1h
4. https://www.glean.com/blog/how-we-analyzed-and-fixed-a-golang-memory-leak
5. https://kirshatrov.com/posts/finding-memory-leak-in-cgo

  В первой [статье](https://www.freecodecamp.org/news/how-i-investigated-memory-leaks-in-go-using-pprof-on-a-large-codebase-4bec4325e192/) обсуждается распространенная проблема утечек памяти, которая часто возникает в приложениях, написанных на языке программирования Go. В нем подчеркивается, что во время выполнения приложения, когда память выделяется для выполнения кода, ему часто не удается освободить память, которая больше не требуется, что приводит к ее накоплению. Это оказывает значительное влияние на общую производительность приложения, а также приводит к перезапуску модуля Kubernetes из-за ограничений оперативной памяти в кластере. 
  Более того, в статье указывается, что, хотя сборка мусора в Go работает эффективно, это не обязательно гарантирует обнаружение утечек памяти. Следовательно, разработчикам крайне важно следовать нескольким рекомендациям, таким как использование defer для обработки очистки ресурсов, избегать циклических ссылок и избегать глобальных переменных, поскольку они остаются в памяти до тех пор, пока приложение не остановится.
В статье также рекомендуется использовать различные инструменты профилирования памяти, такие как pprof и go-torch, для выявления утечек памяти. В заключение в нем подчеркивается, что обнаружение и устранение утечек памяти имеет решающее значение для написания высокопроизводительных приложений Go.
  
  Авторами [второй статьи](https://www.nylas.com/blog/finding-memory-leak-in-go-service-dev/) являются разработчики программного обеспечения из компании nylas, которые запускают свои приложения с помощью Kubernetes, что для меня имеет гораздо больше смысла. И описание их задачи очень похоже на мое, за исключением одной детали: у меня есть код golang, связанный с Facebook faiss framework, движок которого написан на c++. 
Чтобы глубже понять проблему, сборщику мусора были объяснены его схемы и взаимосвязи с кучей. Как и в предыдущей статье, авторы используют proof для поиска узкого места: доступны различные профили для определения причины утечек памяти, включая pprof, который генерирует диаграмму распределения памяти для визуализации использования памяти. В статье также рассматриваются типичные случаи утечек памяти, такие как увеличение глобальных переменных, зависание подпрограмм go и незакрытые файловые потоки. В отличие от предыдущей статьи, чтобы не выполнять постоянное развертывание в рабочем кластере и не воспроизводить утечки памяти локально, авторы предлагают читателям использовать minikube и Tilt. Наконец, статья призывает разработчиков провести аудит своей кодовой базы и выяснить, как произошла утечка памяти. 
  
  В третьей [статье](https://dev.to/googlecloud/finding-and-fixing-memory-leaks-in-go-1k1h) описывается, как была обнаружена и исправлена проблема с утечкой памяти в клиентской библиотеке Google Cloud для Go, которая использует gRPC для подключения к облачным API Google. В статье подробно описывается, как pprof использовался для выявления утечки, как проблема была устранена путем правильного закрытия клиента и как использовался скрипт Bash для редактирования затронутых файлов в образце репозитория Go. В статье также предлагаются способы предотвращения подобных проблем в будущем с помощью улучшенных образцов, документации и библиотек. К сожалению, это не совсем мой случай, но было довольно интересно узнать больше о создании собственного микросервиса от разработчиков Google.
  
  [Четвертая статья](https://www.glean.com/blog/how-we-analyzed-and-fixed-a-golang-memory-leak), автором которой является инженер Шарва Патхак из Glean, в отличие от второй статьи, они используют Go на облачной платформе Google для сервиса с интенсивным использованием памяти. Они заметили неуклонный рост использования памяти, который привел к уничтожению экземпляра из-за превышения лимита памяти. Несмотря на то, что это выглядело как утечка памяти, непрерывное профилирование данных с использованием cloud profiler исключило утечку памяти приложений в качестве причины. Средний размер кучи составлял около 1,5 Г, что указывало на то, что память хранилась где-то во время выполнения Go. Они исключили фрагментацию памяти, но обнаружили потенциальную проблему с взаимодействием Golang и контейнерной среды. Они решили проблему, установив для  переменной среды gaoling другое значение. 
  
  [В последней статье](https://kirshatrov.com/posts/finding-memory-leak-in-cgo) обсуждается, как команда обнаружила и исправила утечку памяти в приложении Go, которое использовало расширение C через cgo. Автор также ссылается на другую статью об утечках памяти в ruby с использованием c-оболочек, это также интересно для поиска. Они использовали классические инструменты, такие как go tool pprof и heaptrack, чтобы найти утечку, но из-за причины другой инструмент - memleak-bpfcc помог им отследить проблему до реальных функций C, и команда смогла это исправить. В статье также подробно описываются изменения, внесенные в Dockerfile для установки libzookeeper из исходного кода с символами отладки, и как команда смогла применить исправление, нашла его и оказала положительное влияние.aa


