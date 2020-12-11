# Тестовые задачи 

## Задача 1

В разных версиях ядра для клонирования `bio` применяются функции `bio_clone` и `bio_clone_fast`. Чем они отличаются с точки зрения работы с `bi_io_vec`?  Предлагается рассматривать версии ядра [5.9](https://elixir.bootlin.com/linux/v5.9.11/source) и [3.10](https://elixir.bootlin.com/linux/v3.10.108/source)

---
### Версия ядра 3.10:
```C
/**
 * 	__bio_clone	-	clone a bio
 * 	@bio: destination bio
 * 	@bio_src: bio to clone
 *
 *	Clone a &bio. Caller will own the returned bio, but not
 *	the actual data it points to. Reference count of returned
 * 	bio will be one.
 */
void __bio_clone(struct bio *bio, struct bio *bio_src)
{
	memcpy(bio->bi_io_vec, bio_src->bi_io_vec, bio_src->bi_max_vecs * (sizeof(struct bio_vec)));
	/*
	 * most users will be overriding ->bi_bdev with a new target,
	 * so we don't set nor calculate new physical/hw segment counts here
	 */
	bio->bi_sector = bio_src->bi_sector;
	bio->bi_bdev = bio_src->bi_bdev;
	bio->bi_flags |= 1 << BIO_CLONED;
	bio->bi_rw = bio_src->bi_rw;
	bio->bi_vcnt = bio_src->bi_vcnt;
	bio->bi_size = bio_src->bi_size;
	bio->bi_idx = bio_src->bi_idx;
}

/**
 *	bio_clone_bioset -	clone a bio
 *	@bio: bio to clone
 *	@gfp_mask: allocation priority
 *	@bs: bio_set to allocate from
 *
 * 	Like __bio_clone, only also allocates the returned bio
 */
struct bio *bio_clone_bioset(struct bio *bio, gfp_t gfp_mask,
			     struct bio_set *bs)
{
	struct bio *b;

	b = bio_alloc_bioset(gfp_mask, bio->bi_max_vecs, bs);
	if (!b)
		return NULL;

	__bio_clone(b, bio);

	if (bio_integrity(bio)) {
		int ret;

		ret = bio_integrity_clone(b, bio, gfp_mask);

		if (ret < 0) {
			bio_put(b);
			return NULL;
		}
	}

	return b;
}

static inline struct bio *bio_clone(struct bio *bio, gfp_t gfp_mask)
{
	return bio_clone_bioset(bio, gfp_mask, fs_bio_set);
}
```

### Версия ядра 5.9

```C

void __bio_crypt_clone(struct bio *dst, struct bio *src, gfp_t gfp_mask)
{
	dst->bi_crypt_context = mempool_alloc(bio_crypt_ctx_pool, gfp_mask);
	*dst->bi_crypt_context = *src->bi_crypt_context;
}

static inline void bio_crypt_clone(struct bio *dst, struct bio *src,
				   gfp_t gfp_mask)
{
	if (bio_has_crypt_ctx(src))
		__bio_crypt_clone(dst, src, gfp_mask);
}

/**
 * bio_clone_blkg_association - clone blkg association from src to dst bio
 * @dst: destination bio
 * @src: source bio
 */
void bio_clone_blkg_association(struct bio *dst, struct bio *src)
{
	if (src->bi_blkg) {
		if (dst->bi_blkg)
			blkg_put(dst->bi_blkg);
		blkg_get(src->bi_blkg);
		dst->bi_blkg = src->bi_blkg;
	}
}

/**
 * 	__bio_clone_fast - clone a bio that shares the original bio's biovec
 * 	@bio: destination bio
 * 	@bio_src: bio to clone
 *
 *	Clone a &bio. Caller will own the returned bio, but not
 *	the actual data it points to. Reference count of returned
 * 	bio will be one.
 *
 * 	Caller must ensure that @bio_src is not freed before @bio.
 */
void __bio_clone_fast(struct bio *bio, struct bio *bio_src)
{
	BUG_ON((bio->bi_pool) && BVEC_POOL_IDX(bio));

	/*
	 * most users will be overriding ->bi_disk with a new target,
	 * so we don't set nor calculate new physical/hw segment counts here
	 */
	bio->bi_disk = bio_src->bi_disk;
	bio->bi_partno = bio_src->bi_partno;
	bio_set_flag(bio, BIO_CLONED);
	if (bio_flagged(bio_src, BIO_THROTTLED))
		bio_set_flag(bio, BIO_THROTTLED);
	bio->bi_opf = bio_src->bi_opf;
	bio->bi_ioprio = bio_src->bi_ioprio;
	bio->bi_write_hint = bio_src->bi_write_hint;
	bio->bi_iter = bio_src->bi_iter;
	bio->bi_io_vec = bio_src->bi_io_vec;

	bio_clone_blkg_association(bio, bio_src);
	blkcg_bio_issue_init(bio);
}

/**
 *	bio_clone_fast - clone a bio that shares the original bio's biovec
 *	@bio: bio to clone
 *	@gfp_mask: allocation priority
 *	@bs: bio_set to allocate from
 *
 * 	Like __bio_clone_fast, only also allocates the returned bio
 */
struct bio *bio_clone_fast(struct bio *bio, gfp_t gfp_mask, struct bio_set *bs)
{
	struct bio *b;

	b = bio_alloc_bioset(gfp_mask, 0, bs);
	if (!b)
		return NULL;
		
	__bio_clone_fast(b, bio);
	bio_crypt_clone(b, bio, gfp_mask);
	
	if (bio_integrity(bio)) {
		int ret;
		ret = bio_integrity_clone(b, bio, gfp_mask);
		if (ret < 0) {
			bio_put(b);
			return NULL;
		}
	}
	return b;
}

```

## Задача 2
