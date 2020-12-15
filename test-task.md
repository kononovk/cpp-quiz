# Тестовые задачи 

## Задача 1

В разных версиях ядра для клонирования `bio` применяются функции `bio_clone` и `bio_clone_fast`. Чем они отличаются с точки зрения работы с `bi_io_vec`?  Предлагается рассматривать версии ядра [5.9](https://elixir.bootlin.com/linux/v5.9.11/source) и [3.10](https://elixir.bootlin.com/linux/v3.10.108/source)
_____

Можно заметить, что `bio_clone` перевыделяет память под `bio_io_vec` и копирует его используя `memcpy` (deep copy):
```C
b = bio_alloc_bioset(...);
memcpy(bio->bi_io_vec, bio_src->bi_io_vec, bio_src->bi_max_vecs * (sizeof(struct bio_vec)));
```
В `bio_clone_fast` используется счетчик ссылок:

`bio_clone_fast`:
```C
b = bio_alloc_bioset(gfp_mask, 0, bs);
__bio_clone_fast(b, bio);
```
`__bio_clone_fast`:
```C
bio->bi_io_vec = bio_src->bi_io_vec;
bio_clone_blkg_association(bio, bio_src);
```
`bio_clone_fast ()` разделяет `bi_io_vec` для клонирования `bio_src` в `bio` (оба указывают на один и тот же список `bi_io_vec`). Далее в `bio_clone_blkg_association` происходит соответствующее изменение счетчика ссылок. Такое улучшение нужно потому что не всем пользователям нужно изменять `bi_io_vec` (если есть гарантия, что `bio_src` не будет освобождена раньше `bio_clone`, то мы можем просто указывать на `bio_src` без глобокого копирования).
