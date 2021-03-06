# Сравнение нескольких способов представления слов для построения языковой модели с использованием градиентного бустинга и нейросетей.

## Решаемая задача

Этот экспериментальный бенчмарк сравнивает разные способы представления слов
в упрощенной задаче построения так называемой [language model](https://en.wikipedia.org/wiki/Language_model).

Language Model - это способ вычислить вероятность заданного фрагмента текста (цепочки слов).
В NLP ранее широко использовались различные варианты построения LM на базе подсчета
употребления N-грамм и сглаживания (smoothing), чтобы скомпенсировать разряженность для N-грамм
большой арности. К примеру, в [NLTK](http://www.nltk.org/) есть реализация [метода Kneser-Ney](http://www.nltk.org/_modules/nltk/probability.html).
Также существует большое количество разных реализаций, отличающихся техническими деталями касательно
хранения больших объемов N-граммной информации и скоростью обработки текста, например [https://github.com/kpu/kenlm](https://github.com/kpu/kenlm).

В последние пару лет широкое применение получили разные варианты neural language models (NLMs),
в которых используются различные нейросетевые архитектуры, включая рекуррентные и сверточные. NLM можно
разбить на две группы: работающие на уровне слов (word-aware NLM) и на уровне символов (character-aware NLM). 
Второй вариант особо интересен тем, что позволяет модели работать с *морфемами*. Это оказывается
чрезвычайно значимо для языков с богатой морфологией. Для некоторых слов потенциально
могут существовать сотни грамматических форм, как в случае прилагательных в русском языке. Даже
очень большой текстовый корпус не дает гарантии, что все варианты форм слов будут включены
в лексикон для word-aware NLM. С другой стороны, морфемные структуры практически всегда
подчиняются регулярным правилам, так что знание основы или базовой формы слова позволяет
генерировать его грамматические и семантические варианты: смелый, смело, осмелеть, осмелев и т.д,
а также определять семантическую связь между однокоренными словами, даже если соответствующие
слова встерчаются впервые и не известна статистика по их контексту их употреблению. Последнее
замечание становится очень важным для областей NLP с динамичным словообразованием, прежде всего это
всякие социальные медиа. 

В рамках бенчмарка подготовлен весь необходимый код для изучения и сравнения обоих вариантов NLM, наряду с другими способами
построения LM - characte-aware и word-aware.

Мы немного упростим нашу Langage Model, поставив задачу в таком виде. Есть N-грамма заранее выбранной длины (см. константу NGRAM_ORDER 
в модуле [DatasetVectorizers](https://github.com/Koziev/WordRepresentations/blob/master/PyModels/DatasetVectorizers.py)).
Если она получена из текстового корпуса (путь к зазипованному
utf-8 файлу прошит в методе _get_corpus_path класса BaseVectorizer), то считаем, что
это валидное сочетание слов и целевое значение y=1. Если же N-грамма получена случайной
заменой одного из слов и такая цепочка не встречается в корпусе, то целевое значение y=0.

Недопустимые N-граммы генерируются в ходе анализа корпуса в том же количестве, что и валидные.
Получается сбалансированный датасет, что облегчает задачу. А отсутствие необходимости ручной разметки
позволяет легко "играться" таким параметром, как количество записей в обучающем датасете.

Таким образом, решается бинарная классификационная задача. Классификатором будет реализация
градиентного бустинга XGBoost и нейросеть, реализованная на Keras, а также несколько
других вариантов на разных языках программирования и DL фреймворках.

Объектом исследования является влияние способа представления слов во входной матрице X
на точность классификации.

Подробный разбор различных подходов в NLP, в том числе для аналогичных задач,
можно найти в статье: [A Primer on Neural Network Models for Natural Language Processing](https://arxiv.org/pdf/1510.00726.pdf).

## Варианты представления слов

Для XGBoost проверяются следующие варианты:

* **w2v** - используем заренее обученную word2vec, склеивая векторы слов в один длинный вектор  

* **w2v_tags** - расширение w2v-представления, дополнительные разряды итогового вектора
получаются из морфологических тегов слов. Получается что-то типа [Factored Language Model](http://melodi.ee.washington.edu/people/bilmes/mypapers/hlt03.pdf)  

* **sdr** - sparse distributed representations слов, получаемые факторизацией матрицы word2vector модели  

* **random_bitvector** - каждому слову приписывается случайный бинарный вектор фиксированной длины с заданной пропорцией 0/1  

* **bc** - в качестве репрезентаций используются векторы, созданные в результате работы brown clustering (см. описание https://en.wikipedia.org/wiki/Brown_clustering и реализацию https://github.com/percyliang/brown-cluster)  

* **chars** - каждое слово кодируется как цепочка из 1-hot репрезентаций символов  

* **hashing_trick** - используется hashing trick для кодирования слов ограниченным числом битов индекса (см. описание https://en.wikipedia.org/wiki/Feature_hashing и реализацию https://radimrehurek.com/gensim/corpora/hashdictionary.html)  

* **ae** - векторы слов получаются как активации на внутреннем слое автоэнкодера. Обучение
автоэнкодера и получение векторов слов выполняется скриптом [word_autoencoder3.py](https://github.com/Koziev/WordRepresentations/blob/master/PyModels/WordAutoEncoders/word_autoencoder3.py)
в папке PyModels/WordAutoEncoders.

Для нейросетей доступны два дополнительных способа представления, которые с помощью
слоя Embedding (или аналогичного для используемой библиотеки deep learning) преобразуются
в некоторое векторное представление.

* **word_indeces** - составляется лексикон, каждому слову присваивается уникальный целочисленный
индекс. Таким образом, 3-грамма представляется тройкой целых чисел.  

* **char_indeces** - составляется алфавит, каждому символу присваивается уникальный
целочисленный индекс. Далее цепочки индексов символов дополняются пробельным индексом
до одинаковой длины (максимальная длина слова), и получившаяся цепочка индексов символов
возвращается в качестве представления n-граммы. С помощью слоя встраивания (Embedding в
Keras или аналогичного) символы преобразуются в dense vector форму, с которой работает
остальная часть нейросети. Получается вариант [Character-Aware Neural Language Model](https://arxiv.org/abs/1508.06615).

## Модули и решатели

Для решения задачи использовались различные библиотеки машинного обучения, нейросетевые
фреймворки и библиотеки для матричных вычислений, в том числе:

[Keras](https://keras.io/) (с Theano backend)  
[Lasagne](https://github.com/Lasagne/Lasagne) (Theano)  
[nolearn](https://github.com/dnouri/nolearn) (Theano)  
[TensorFlow](https://www.tensorflow.org/)  


**Решения на Python:**  
[PyModels/wr_xgboost.py](https://github.com/Koziev/WordRepresentations/blob/master/PyModels/wr_xgboost.py) - решатель на базе [XGBoost](http://xgboost.readthedocs.io/en/latest/python/python_api.html)  
[PyModels/wr_catboost.py](https://github.com/Koziev/WordRepresentations/blob/master/PyModels/wr_catboost.py) - решатель на базе [CatBoost](https://github.com/catboost/catboost) по индексам слов, использующий возможность указать индексы категориальных признаков в датасете, чтобы бустер самостоятельно учел их при тренировке  
[PyModels/wr_keras.py](https://github.com/Koziev/WordRepresentations/blob/master/PyModels/wr_keras.py) - решатель на базе feed forward нейросетки, реализованной на [Keras](https://keras.io/)  
[PyModels/wr_keras_sdr2.py](https://github.com/Koziev/WordRepresentations/blob/master/PyModels/wr_keras_sdr2.py) - Отдельное решение для проверки sparse distributed representation слов большой размерности (от 1024) с порционной генерацией матриц для обучения и валидации, на Keras.
[PyModels/wr_lasagne.py](https://github.com/Koziev/WordRepresentations/blob/master/PyModels/wr_lasagne.py) - решатель на базе feed forward нейросетки, реализованной на [Lasagne](https://github.com/Lasagne/Lasagne) (Theano)  
[PyModels/wr_nolearn.py](https://github.com/Koziev/WordRepresentations/blob/master/PyModels/wr_nolearn.py) - решатель на базе feed forward нейросетки, реализованной на [nolearn](https://github.com/dnouri/nolearn)+Lasagne (Theano)  
[PyModels/wr_tensorflow.py](https://github.com/Koziev/WordRepresentations/blob/master/PyModels/wr_tensorflow.py) - решатель на базе feed forward нейросетки, реализованной на [TensorFlow](https://www.tensorflow.org/)  

**Решения на C#:**  
[CSharpModels/WithAccordNet/Program.cs](https://github.com/Koziev/WordRepresentations/blob/master/CSharpModels/WithAccordNet/Program.cs) - решатель на базе feed forward сетки [Accord.NET](http://accord-framework.net/) (C#, проект для VS 2015)  
[CSharpModels/MyBaseline/Program.cs](https://github.com/Koziev/WordRepresentations/blob/master/CSharpModels/MyBaseline/Program.cs) - решение на базе моей реализации vanilla MLP (C#, проект для VS 2015)  

**Решения на C++:**  
[CXXModels/TinyDNN_Model/TinyDNN_Model.cpp](https://github.com/Koziev/WordRepresentations/blob/master/CXXModels/TinyDNN_Model/TinyDNN_Model.cpp) - решатель на базе MLP, реализованного средствами библиотеки [tiny-dnn](https://github.com/tiny-dnn) (C++, проект для VS 2015)  
[CXXModels/Singa_Model/alexnet.cc](https://github.com/Koziev/WordRepresentations/blob/master/CXXModels/Singa_Model/alexnet.cc) - решатель на базе нейросетки, реализованной средствами [Apache.SINGA](https://singa.incubator.apache.org/en/index.html) (C++, проект для VS 2015)  
[CXXModels/OpenNN_Model/main.cpp](https://github.com/Koziev/WordRepresentations/blob/master/CXXModels/OpenNN_Model/main.cpp) - решатель на базе нейросетки, реализованной средствами [OpenNN](http://www.opennn.net/) (C++, проект для VS 2015)  

**Решения на Java**
[JavaModels/WithDL4J/src/main/java/WordRepresentationsTest.java](https://github.com/Koziev/WordRepresentations/blob/master/JavaModels/WithDL4J/src/main/java/WordRepresentationsTest.java) - решатель на базе MLP, реализованного средствами библиотеки [deeplearning4j](https://deeplearning4j.org/)  

**Внутренние классы и инструменты:**  
PyModels/CorpusReaders.py - классы для чтения строк из текстовых корпусов разных форматов (zipped|plain txt)
PyModels/DatasetVectorizers.py - векторизаторы датасета и фабрика для удобного выбора векторизатора по его условному названию  
PyModels/store_dataset_file.py - генерация датасета и его сохранение для C# и C++ моделей  


## Нейросетевые архитектуры

Для нейросеточного решения на базе Keras реализовано 2 архитектуры.
**MLP** - простая feed forward нейросетка с полносвязанными слоями.
**ConvNet** - сетка, в которой используются [сверточные слои](https://keras.io/layers/convolutional/).

Переключение архитектур выполняется настроечным параметром NET_ARCH в файле wr_keras.py.

Можно заметить, что сверточный вариант зачастую дает более точное решение, но соответствующие
модели имеют больший variance, попросту говоря - итоговая точность сильно скачет для
нескольких моделей с одинаковыми параметрами.

## Нейросетевые архитектуры автоэнкодера для char embedding представлений

Подробное описание архитектуры и числовые результаты экспериментов можно найти тут: https://kelijah.livejournal.com/224925.html.


## Формат текстового корпуса

Все варианты бенчмарка используют тектовый файл в кодировке utf-8 для получения
списков N-грамм. Предполагается, что разбивка текста на слова и приведение к нижнему
регистру выполнены заранее сторонним кодом. Поэтому скрипты читают строки из этого файла,
разбивают их на слова по пробелам.

Для удобства воспроизведения экспериментов я поместил в папку data сжатую выдержку
из большого корпуса размером 100 тысяч строк. Этого достаточно для повторения всех
замеров. Чтобы не перегружать репозиторий большим файлом, он зазипован. Метод BaseVectorizer._load_ngrams
сам распаковывает его содержимое на лету, поэтому никаких ручных манипуляций делать не надо.


## Грамматический словарь

В варианте векторизации w2v_tags к w2v-векторам слов добавляются морфологические признаки.
Эти признаки для каждого слова берутся из упакованного файла word2tags_bin.zip в подкаталоге data,
который получен конвертацией моего грамматического словаря (http://solarix.ru/sql-dictionary-sdk.shtml).
Результат конвертации тянет на 165 Мб, что многовато для git репозитория. Поэтому я 
зазиповал получившимйся файл грамматического словаря, а соответствующий класс
векторизации датасета сам распаковывает его на лету во время работы.

Обратите внимание, что из-за омонимии грамматических форм многие слова имеют более
одного варианта набора тегов. Например, слово 'мишки' может быть формой единственного
или множественного числа соответственно в родительном или именительном падеже. При формировании
наборов тегов для слов я объединяю теги омонимов.


## Получение sparse distributed representation для слов

Используется двухэтапный процесс.
Сначала получаем w2v векторы слов. Небольшой скрипт на питоне выгружает векторы
в текстовый файл в csv формате.

Затем с помощью утилиты https://github.com/mfaruqui/sparse-coding выполняется команда:  

./nonneg.o input_vectors.txt 16 0.2 1e-5 4 out_vecs.txt

Генерируется файл out_vecs.txt, содержащий SDR слов с указанными параметрами: размер=16*длина вектора w2v, заполнение=0.2
Этот файл грузится классом SDR_Vectorizer.


## Дополнительные подробности:  

https://habrahabr.ru/post/335838/  
http://kelijah.livejournal.com/217608.html  



## Baseline  

В качестве проверки используем запоминание N-грамм из тренировочного набора и
поиск среди запомненных N-грамм для проверочного набора. Небольшой размер тренировочного
набора приводит к тому, что на проверочном наборе модель ведет себя просто как рандомный
угадыватель, давая точность 0.50.


## Текущие результаты

Основная таблица результатов доступна по [ссылке](https://github.com/Koziev/WordRepresentations/blob/master/results/results.txt).
Остальные перечисленные далее результаты получены для набора из 1,000,000 3-грамм.

### XGBoost

Для решателя на базе XGBoost (wr_xgboost.py) получены следующие результаты.

Для датасета, содержащего 1,000,000 триграмм:
w2v_tags (word2vector + morph tags) ==> 0.81  
w2v (word2vector dim=32) ==> 0.80  
sdr (sparse distributed representations, длина 512, 20% единиц) ==> 0.78
bc (brown clustering) ==> 0.70  
chars (char one-hot encoding) ==> 0.71  
random_bitvector (random bit vectors, доля единиц 16%) ==> 0.696  
hashing_trick (hashing trick with 32,000 slots) ==> 0.64  

Для решателя на базе Lasagne MLP:

accuracy=0.71  

Для решателя на базе nolearn+lasagne MLP:

accuracy=0.64

Для решателя на базе tiny-dnn MLP:

accuracy=0.58

Для решателя на базе Accord.NET feed forward neural net:

word2vector ==> 0.68  

Для решателя на базе CatBoost по категориальным признакам "индекс слова":

accuracy=0.50  

Для решателя на базе Apache.SINGA MLP:  

accuracy=0.74  

Baseline решение - запоминание N-грамм из тренировочного датасета

accuracy=0.50


