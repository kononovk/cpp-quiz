# Тестовые задачи 

## Задача 1

В разных версиях ядра для клонирования `bio` применяются функции `bio_clone` и `bio_clone_fast`. Чем они отличаются с точки зрения работы с `bi_io_vec`?  Предлагается рассматривать версии ядра [5.9](https://elixir.bootlin.com/linux/v5.9.11/source) и [3.10](https://elixir.bootlin.com/linux/v3.10.108/source)

---
### Версия ядра 3.10:
```C
#if defined(CONFIG_BLK_DEV_INTEGRITY)
#define bio_integrity(bio) (bio->bi_integrity != NULL)
#else /* CONFIG_BLK_DEV_INTEGRITY */
static inline int bio_integrity(struct bio *bio)
{
	return 0;
}
#endif

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
#if defined(CONFIG_BLK_DEV_INTEGRITY)
static inline struct bio_integrity_payload *bio_integrity(struct bio *bio)
{
	if (bio->bi_opf & REQ_INTEGRITY)
		return bio->bi_integrity;
	return NULL;
}
#else
static inline void *bio_integrity(struct bio *bio)
{
	return NULL;
}
#endif

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
    // bio->integrity != NULL
	if (bio_integrity(bio)) {
		int ret;
		ret = bio_integrity_clone(b, bio, gfp_mask);  // аллокация памяти под bio_integrity_payload
		// если код возврата < 0 (ошибка) -> уменьшаем ref_count на 1
		if (ret < 0) {                                
			bio_put(b);
			return NULL;
		}
	}
	return b;
}
```
_____

Можно заметить, что `bio_clone` перевыделяет память под странички и копирует их используя `memcpy` (deep copy):
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
`bio_clone_blkg_association`:
```C
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
```


## Задача 2
```C
void func(struct bio *bio1, int len, struct bio_set *bs) 
{ 
    struct bio* bio2 = bio_split(bio1, len, GFP_NOIO, bs); 
    bio_chain(bio1, bio2); 
    submit_bio(bio2); 
    submit_bio(bio1); 
} 
```
Прочитать код функций `bio_split` и `bio_chain` в ядре версии [5.9](https://elixir.bootlin.com/linux/v5.9.11/source) и ответить на следующий вопрос: Одназначно ли определен порядок вызова функций `bio1->bi_end_io` и `bio2->bi_end_io`.

```C

/**
 * bio_split - split a bio
 * @bio:	bio to split
 * @sectors:	number of sectors to split from the front of @bio
 * @gfp:	gfp mask
 * @bs:		bio set to allocate from
 *
 * Allocates and returns a new bio which represents @sectors from the start of
 * @bio, and updates @bio to represent the remaining sectors.
 *
 * Unless this is a discard request the newly allocated bio will point
 * to @bio's bi_io_vec. It is the caller's responsibility to ensure that
 * neither @bio nor @bs are freed before the split bio.
 */
struct bio *bio_split(struct bio *bio, int sectors,
		      gfp_t gfp, struct bio_set *bs)
{
	struct bio *split;

	BUG_ON(sectors <= 0);
	BUG_ON(sectors >= bio_sectors(bio));

	/* Zone append commands cannot be split */
	if (WARN_ON_ONCE(bio_op(bio) == REQ_OP_ZONE_APPEND))
		return NULL;

	split = bio_clone_fast(bio, gfp, bs);
	if (!split)
		return NULL;

	split->bi_iter.bi_size = sectors << 9;

	if (bio_integrity(split))
		bio_integrity_trim(split);

	bio_advance(bio, split->bi_iter.bi_size);

	if (bio_flagged(bio, BIO_TRACE_COMPLETION))
		bio_set_flag(split, BIO_TRACE_COMPLETION);

	return split;
}


/**
 * bio_chain - chain bio completions
 * @bio: the target bio
 * @parent: the @bio's parent bio
 *
 * The caller won't have a bi_end_io called when @bio completes - instead,
 * @parent's bi_end_io won't be called until both @parent and @bio have
 * completed; the chained bio will also be freed when it completes.
 *
 * The caller must not set bi_private or bi_end_io in @bio.
 */
void bio_chain(struct bio *bio, struct bio *parent)
{
	BUG_ON(bio->bi_private || bio->bi_end_io);

	bio->bi_private = parent;
	bio->bi_end_io	= bio_chain_endio;
	bio_inc_remaining(parent);
}

/**
 * submit_bio - submit a bio to the block device layer for I/O
 * @bio: The &struct bio which describes the I/O
 *
 * submit_bio() is used to submit I/O requests to block devices.  It is passed a
 * fully set up &struct bio that describes the I/O that needs to be done.  The
 * bio will be send to the device described by the bi_disk and bi_partno fields.
 *
 * The success/failure status of the request, along with notification of
 * completion, is delivered asynchronously through the ->bi_end_io() callback
 * in @bio.  The bio must NOT be touched by thecaller until ->bi_end_io() has
 * been called.
 */
blk_qc_t submit_bio(struct bio *bio)
{
	if (blkcg_punt_bio_submit(bio))
		return BLK_QC_T_NONE;

	/*
	 * If it's a regular read/write or a barrier with data attached,
	 * go through the normal accounting stuff before submission.
	 */
	if (bio_has_data(bio)) {
		unsigned int count;

		if (unlikely(bio_op(bio) == REQ_OP_WRITE_SAME))
			count = queue_logical_block_size(bio->bi_disk->queue) >> 9;
		else
			count = bio_sectors(bio);

		if (op_is_write(bio_op(bio))) {
			count_vm_events(PGPGOUT, count);
		} else {
			task_io_account_read(bio->bi_iter.bi_size);
			count_vm_events(PGPGIN, count);
		}

		if (unlikely(block_dump)) {
			char b[BDEVNAME_SIZE];
			printk(KERN_DEBUG "%s(%d): %s block %Lu on %s (%u sectors)\n",
			current->comm, task_pid_nr(current),
				op_is_write(bio_op(bio)) ? "WRITE" : "READ",
				(unsigned long long)bio->bi_iter.bi_sector,
				bio_devname(bio, b), count);
		}
	}

	/*
	 * If we're reading data that is part of the userspace workingset, count
	 * submission time as memory stall.  When the device is congested, or
	 * the submitting cgroup IO-throttled, submission can be a significant
	 * part of overall IO time.
	 */
	if (unlikely(bio_op(bio) == REQ_OP_READ `&&` bio_flagged(bio, BIO_WORKINGSET))) {
		unsigned long pflags;
		blk_qc_t ret;
		psi_memstall_enter(&pflags);
		ret = submit_bio_noacct(bio);
		psi_memstall_leave(&pflags);
		return ret;
	}
	return submit_bio_noacct(bio);
}
```
