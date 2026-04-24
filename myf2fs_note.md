# F2FS Modification
<!-- ![myf2fs](https://hackmd.io/_uploads/rJikG8s2We.png) -->
[Link](https://app.diagrams.net/?src=about#G14VSKaUAX2heYxhCLAHq66LBSfl1w-xfR#%7B%22pageId%22%3A%22StU38bJLJpKlbOlZgCsN%22%7D)
## super.c
### struct file_system_type
``` c
static struct file_system_type f2fs_fs_type = {
	.owner		= THIS_MODULE,
	.name		= "myf2fs",    # change name
	.mount		= f2fs_mount,
	.kill_sb	= kill_f2fs_super,
	.fs_flags	= FS_REQUIRES_DEV | FS_ALLOW_IDMAP,
};
MODULE_ALIAS_FS("myf2fs");   # change name
```


### f2fs_mount()
`mount_nodev` (Mount No Device) 是 Linux 提供給那些沒有實體硬碟的虛擬檔案系統使用的掛載函式，例如 proc (/proc)、sysfs (/sys) 或是 tmpfs
- 功能
	- 不檢查 /dev/dax13.0 
	- 建立一個乾淨的 Superblock (sb)
	- 為它是 nodev，所以它會把 `sb->s_bdev` 設定為 `NULL`
``` c
static struct dentry *f2fs_mount(struct file_system_type *fs_type, int flags,
			const char *dev_name, void *data)
{
    /* --- NEW: 攔截 CXL DAX 裝置 --- */
    if (dev_name && strstr(dev_name, "/dev/dax")) {
        // 因為是字元裝置，我們改用 mount_nodev 來繞過 VFS 的區塊檢查
        // 並將 f2fs_fill_super 作為 callback 傳入
        return mount_nodev(fs_type, flags, data, f2fs_fill_super);
    }
    /* ------------------------------ */
	
    return mount_bdev(fs_type, flags, dev_name, data, f2fs_fill_super);
}
```

### f2fs_fill_super()
執行 mount 時會 call 到這個 function，功能是將硬碟中的資料轉換成 Linux kernel 裡的資料結構。如果它執行成功，掛載點就會出現。

``` c
	... 

    /* --- NEW: 1. CXL Native 初始化與 Block Size 手動設定 --- */
    // 當我們用 mount_nodev 掛載時，sb->s_bdev 會是 NULL
    if (sb->s_bdev == NULL) {
        sbi->is_cxl_dax = true;
        
        // 手動設定 Block Size
        sb->s_blocksize = F2FS_BLKSIZE;
        sb->s_blocksize_bits = F2FS_BLKSIZE_BITS;   // 12 (4096 = 2^12)

	// 1. 從 mount options (傳入的 data 字串) 解析實體位址與大小
        if (data) {
            char *p;
            p = strstr((char *)data, "cxl_phys=");
            if (p) sbi->cxl_phys_addr = simple_strtoull(p + 9, NULL, 16);
            
            p = strstr((char *)data, "cxl_size=");
            if (p) sbi->cxl_size = simple_strtoull(p + 9, NULL, 10);
        }

        ...

        // 3. 正式將 CXL 實體位址映射到 Kernel 虛擬位址 
        sbi->cxl_base_addr = memremap(sbi->cxl_phys_addr, sbi->cxl_size, MEMREMAP_WB);
        
        if (!sbi->cxl_base_addr) {
            f2fs_err(sbi, "Failed to memremap CXL memory at 0x%llx", (unsigned long long)sbi->cxl_phys_addr);
            err = -ENOMEM;
            goto free_sbi;
        }
        
        f2fs_info(sbi, "CXL Native DAX mode enabled. Bypassing Block Layer.");
    } else {
        sbi->is_cxl_dax = false;
    }

	...

	/* --- NEW: 3. 攔截 Superblock 讀取，改由記憶體直接映射 --- */
    if (sbi->is_cxl_dax) {
        // 直接從 CXL 實體記憶體指標加上 Offset (1024 bytes) 讀取 Superblock
        raw_super = (struct f2fs_super_block *)((char *)sbi->cxl_base_addr + F2FS_SUPER_OFFSET);
        
        if (le32_to_cpu(raw_super->magic) != F2FS_SUPER_MAGIC) {
            f2fs_err(sbi, "CXL Memory does not contain a valid F2FS superblock!");
            err = -EINVAL;
            goto free_sbi;
        }
        
        valid_super_block = 1; // 假設直接有效
        err = 0;
    } else {
        err = read_raw_super_block(sbi, &raw_super, &valid_super_block, &recovery);
    }
    /* -------------------------------------------------------- */
```


### static int parse_options()
mount 時會將 cxl base address and size 用參數的方式傳進來  
這裡新增 parsing 的邏輯，並存入 sbi struct
``` c
static int parse_options(struct super_block *sb, char *options, bool is_remount)
{
	struct f2fs_sb_info *sbi = F2FS_SB(sb);
	substring_t args[MAX_OPT_ARGS];
	
	...

	while ((p = strsep(&options, ",")) != NULL) {
            int token;

            args[0].to = args[0].from = NULL;
            token = match_token(p, f2fs_tokens, args);

	    switch (token) {
                /* --- CXL DAX NATIVE MOD: 讀取並轉換 64 位元位址與大小 --- */
                case Opt_cxl_phys:
                    name = match_strdup(&args[0]);
                    if (!name)
                        return -ENOMEM;
                    sbi->is_cxl_dax = true;
                    sbi->cxl_phys_addr = simple_strtoull(name, NULL, 16); // string to ULL
                    kfree(name);
                    break;

                case Opt_cxl_size:
                    name = match_strdup(&args[0]);
                    if (!name)
                        return -ENOMEM;
                    sbi->cxl_size = simple_strtoull(name, NULL, 10); // 10進位解析
                    kfree(name);
                    break;
                /* ---------------------------------------------------- */
	...
```


### static void default_options()
bypass some hardware check  (e.g., support discard or not)
``` c
static void default_options(struct f2fs_sb_info *sbi, bool remount)
{
    /* init some FS parameters */
    if (!remount) {
        set_opt(sbi, READ_EXTENT_CACHE);
        clear_opt(sbi, DISABLE_CHECKPOINT);

        /* --- CXL DAX NATIVE MOD: bypass hardware test --- */
        if (sbi->is_cxl_dax) {
            set_opt(sbi, DISCARD);
        } else {
            // 只有非 CXL 模式才去讀取 s_bdev 硬體資訊
            if (f2fs_hw_support_discard(sbi) || f2fs_hw_should_discard(sbi))
                set_opt(sbi, DISCARD);
        }
        /* ---------------------------------------------------------- */
    }
    ...    
}

```



### static int f2fs_scan_devices()
開啟 multi-device 中的每一個 device ，並讀取他們的硬體資訊  
因為原本 F2FS 預設所有 device 都是 block device，所以要額外處理 CXL DAX device 的邏輯
算每個 device 的 segment 數量、open device
``` c
#define FDEV(i)				(sbi->devs[i])
#define RDEV(i)				(raw_super->devs[i])

static int f2fs_scan_devices(struct f2fs_sb_info *sbi)
{
	struct f2fs_super_block *raw_super = F2FS_RAW_SUPER(sbi);
	unsigned int max_devices = MAX_DEVICES;
	unsigned int logical_blksize;
	blk_mode_t mode = sb_open_mode(sbi->sb->s_flags);
	int i;

	...

	/* --- CXL DAX NATIVE MOD: Device 0 是 CXL 時，沒有 s_bdev 邏輯尺寸，預設為 4096 --- */
        if (sbi->is_cxl_dax)
            logical_blksize = F2FS_BLKSIZE; 
        else
            logical_blksize = bdev_logical_block_size(sbi->sb->s_bdev);
        /* ----------------------------------------------------------------------- */
	
	sbi->aligned_blksize = true;

	for (i = 0; i < max_devices; i++) {

	    /* --- CXL DAX NATIVE MOD: 攔截 Device 0 (CXL DAX) --- */
            if (sbi->is_cxl_dax && i == 0) {
                FDEV(i).bdev_handle = NULL; // CXL has no block_device structure
                FDEV(i).bdev = NULL; // CXL has no block_device structure

                memcpy(FDEV(i).path, RDEV(i).path, MAX_PATH_LEN);  // copy device name e.g., /dev/dax13.0
                FDEV(i).total_segments = le32_to_cpu(RDEV(i).total_segments);
                FDEV(i).start_blk = 0;   // block start from 0
                // 找出這塊空間的 end block idx
                // 算法為 總 segment * (2MB / 4KB)  (total_segments 為給 main area 使用的 seg 數)
                // -1: for idx start from 0
                // + le32_to_cpu(raw_super->segment0_blkaddr): 加上被 metadata 佔用的 block 數量
                FDEV(i).end_blk = FDEV(i).start_blk +
                    (FDEV(i).total_segments << sbi->log_blocks_per_seg) - 1 +
                    le32_to_cpu(raw_super->segment0_blkaddr);

                sbi->s_ndevs = i + 1;   // s_ndevs: number of register device
                continue; 
            }
            /* ----------------------------------------------- */
		...
	}
}
```











### static void f2fs_put_super()
在 unmount 時 VFS 會 call 的 function，主要功能是歸還所有跟系統借來的資源  
``` c
/* --- CXL DAX NATIVE MOD: 在釋放 sbi 之前，解除 memory mapping --- */
if (sbi->is_cxl_dax && sbi->cxl_base_addr) {
    memunmap(sbi->cxl_base_addr);
    sbi->cxl_base_addr = NULL;
    f2fs_info(sbi, "CXL Native Memory unmapped.");
}


/* --- CXL DAX NATIVE MOD: raw_super is not allocated by kmalloc() --- */
if (!sbi->is_cxl_dax)
    kfree(sbi->raw_super);
/* ----------------------------------------------------------- */
```
### int f2fs_commit_super()





## f2fs.h
### struct f2fs_sb_info
``` c
struct f2fs_sb_info {
    /* --- NEW: CXL DAX Native Support --- */
    bool is_cxl_dax;                   /* 標記是否為 CXL DAX 模式 */
    void *cxl_base_addr;               /* CXL 實體記憶體的 Kernel 虛擬映射指標 */
  
    phys_addr_t cxl_phys_addr;         /* CXL 記憶體的硬體實體起點位址 */
    size_t cxl_size;                   /* CXL 記憶體的總大小 (Bytes) */
    /* ------------------------------- */
    ...
```

### missing macro
``` c
#ifndef F2FS_IO_SIZE_BITS
    #define F2FS_IO_SIZE_BITS(sbi)  (0)
    #define F2FS_IO_SIZE(sbi)       (1)
    #define F2FS_IO_SIZE_KB(sbi)    (4)
#endif

#ifndef F2FS_IO_SIZE_MASK
    #define F2FS_IO_SIZE_MASK(sbi) (F2FS_IO_SIZE(sbi) - 1)
#endif

/* CXL DAX */
#ifndef F2FS_IO_ALIGNED
    #define F2FS_IO_ALIGNED(sbi) (0)
#endif
```


## xattr.c
### int f2fs_init_xattr_caches()


## data.c
### void f2fs_submit_read_bio()
``` c
void f2fs_submit_read_bio(struct f2fs_sb_info *sbi, struct bio *bio,
				 enum page_type type)
{
    WARN_ON_ONCE(!is_read_io(bio_op(bio)));

    /* --- CXL DAX NATIVE MOD:  intercept Device 0 --- */
    if (sbi->is_cxl_dax && bio->bi_bdev == NULL) {
        struct bio_vec bvl;
        struct bvec_iter iter;

        // 正確使用 bvec_iter 來獲取每個 page 的真實 sector 偏移量
        bio_for_each_segment(bvl, bio, iter) {
            struct page *page = bvl.bv_page;
            void *cxl_addr = (char *)sbi->cxl_base_addr + (iter.bi_sector << 9); 
            void *kaddr = kmap_atomic(page);

            memcpy(kaddr, cxl_addr, bvl.bv_len);

            kunmap_atomic(kaddr);
            flush_dcache_page(page);
            SetPageUptodate(page);
        }
        bio->bi_status = BLK_STS_OK; // 標記為成功
        bio_endio(bio); // 直接標記 bio 為結束
        return;
    }
    /* --------------------------------------------- */

    trace_f2fs_submit_read_bio(sbi->sb, type, bio);

    iostat_update_submit_ctx(bio, type);
    submit_bio(bio);
}
```
### static void f2fs_submit_write_bio()
``` c
static void f2fs_submit_write_bio(struct f2fs_sb_info *sbi, struct bio *bio,
				  enum page_type type)
{
    WARN_ON_ONCE(is_read_io(bio_op(bio)));

    /* --- CXL DAX NATIVE MOD:  intercept Device 0 --- */
    if (sbi->is_cxl_dax && bio->bi_bdev == NULL) {
        struct bio_vec bvl;
        struct bvec_iter iter;
        
        bio_for_each_segment(bvl, bio, iter) {
            struct page *page = bvl.bv_page;
            void *cxl_addr = (char *)sbi->cxl_base_addr + (iter.bi_sector << 9);
            void *kaddr = kmap_atomic(page);
            
            memcpy_flushcache(cxl_addr, kaddr, bvl.bv_len);
            
            kunmap_atomic(kaddr);
        }
        bio->bi_status = BLK_STS_OK;
        bio_endio(bio); 
        return;
    }
    /* --------------------------------------------- */

    if (type == DATA || type == NODE) {
        if (f2fs_lfs_mode(sbi) && current->plug)
            blk_finish_plug(current->plug);

        if (F2FS_IO_ALIGNED(sbi)) {
            f2fs_align_write_bio(sbi, bio);
            /*
             * In the NODE case, we lose next block address chain.
             * So, we need to do checkpoint in f2fs_sync_file.
             */
            if (type == NODE)
                set_sbi_flag(sbi, SBI_NEED_CP);
        }
    }

    trace_f2fs_submit_write_bio(sbi->sb, type, bio);
    iostat_update_submit_ctx(bio, type);
    submit_bio(bio);
}

```
### int f2fs_submit_page_bio()



## segment.c
### int f2fs_start_discard_thread()
F2FS 準備啟動「背景垃圾回收（Discard/TRIM）執行緒」，它正試圖去檢查每個裝置的 Queue 屬性，看看是否支援 Discard
CXL 不支援 discard 所以 bypass 檢查 
``` c
int f2fs_start_discard_thread(struct f2fs_sb_info *sbi)
{
    dev_t dev;
    struct discard_cmd_control *dcc = SM_I(sbi)->dcc_info;
    int err = 0;

    /* --- CXL DAX NATIVE MOD: 安全獲取設備號碼 --- */
    if (sbi->is_cxl_dax || !sbi->sb->s_bdev)
        dev = sbi->sb->s_dev;
    else
        dev = sbi->sb->s_bdev->bd_dev;

    /* * --- CXL DAX NATIVE MOD: 繞過危險的巨集檢查 ---
     * CXL 本身不支援傳統 Discard，但 ZNS 支援。
     * 我們強行讓 Thread 啟動，因為後面的 ZNS 裝置需要它。
     */
    if (sbi->is_cxl_dax) {
        // 在 CXL 模式下，我們略過 bdev_max_discard_sectors 檢查
        // 只要 ZNS 存在，我們就允許啟動
    } else if (!f2fs_realtime_discard_enable(sbi)) {
        return 0;
    }

    /* 檢查 dcc 是否存在，防止 NULL 存取 */
    if (!dcc)
        return 0;

    if (dcc->f2fs_issue_discard)
        return 0;

    dcc->f2fs_issue_discard = kthread_run(issue_discard_thread, sbi,
                "f2fs_discard-%u:%u", MAJOR(dev), MINOR(dev));
    
    if (IS_ERR(dcc->f2fs_issue_discard)) {
        err = PTR_ERR(dcc->f2fs_issue_discard);
        dcc->f2fs_issue_discard = NULL;
    }

    return err;
}
```

### static int __submit_discard_cmd()
F2FS 發送 TRIM 指令給硬碟的 function
參數 `struct discard_cmd *dc` 為 discard 內容清單，包含 target 哪顆硬碟、start addr、size
將此清單打包成 bio request，將 BIO 標上 REQ_OP_DISCARD 標籤，然後呼叫 submit_bio() 送進底層的硬碟 driver
把這個指令的狀態改成 D_SUBMIT（已送出），並放進等待佇列中，等硬碟回報完成
modification: 攔截針對 CXL 的 Discard 指令，在沒有實際對硬體做任何事的情況下，透過修改內部狀態，讓 F2FS 以為「硬碟已經順利做完 TRIM 了」。
``` c
static int __submit_discard_cmd(struct f2fs_sb_info *sbi,
				struct discard_policy *dpolicy,
				struct discard_cmd *dc, int *issued)
{
    struct block_device *bdev = dc->bdev;
    unsigned int max_discard_blocks;
    struct discard_cmd_control *dcc = SM_I(sbi)->dcc_info;
    struct list_head *wait_list = (dpolicy->type == DPOLICY_FSTRIM) ?
                    &(dcc->fstrim_list) : &(dcc->wait_list);
    blk_opf_t flag = dpolicy->sync ? REQ_SYNC : 0;
    block_t lstart, start, len, total_len;
    int err = 0;

    if (dc->state != D_PREP)
        return 0;

    if (is_sbi_flag_set(sbi, SBI_NEED_FSCK))
        return 0;

    /* --- CXL DAX NATIVE MOD: 攔截 Device 0 (CXL) 的 Discard 指令 --- */
    if (sbi->is_cxl_dax && bdev == NULL) {
        unsigned long flags;

        trace_f2fs_issue_discard(bdev, dc->di.start, dc->di.len);

        /* 1. 標記為完成 (D_DONE)，這樣 wait_for_completion 就會直接略過等待 */
        spin_lock_irqsave(&dc->lock, flags);
        dc->state = D_DONE;
        spin_unlock_irqrestore(&dc->lock, flags);

        /* 2. 維持正確的計數，防止 umount 時卡死 */
        atomic_inc(&dcc->queued_discard);
        dc->queued++;

        /* 3. 移交給 wait_list，讓 F2FS 的回收機制處理剩下的釋放動作 */
        list_move_tail(&dc->list, wait_list);

        /* 4. 更新 SIT Bitmap */
        __check_sit_bitmap(sbi, dc->di.lstart, dc->di.lstart + dc->di.len);

        (*issued)++;
        return 0;
    }

    /* 在 CXL 檢查之後才讀取 bdev 屬性，確保絕對安全 */
    max_discard_blocks = SECTOR_TO_BLOCK(bdev_max_discard_sectors(bdev));
    /* ----------------------------------------------------------- */
```


## checkpoint.c
### int f2fs_start_ckpt_thread()

## sysfs.c
所有的 `f2fs` 都要改成 `myf2fs`

# compile
``` bash
make  # 產出 myf2fs.ko

sudo insmod myf2fs.ko

lsmod | grep f2fs  # 應該要看到 myf2fs
```

若 `insmod` 失敗可看下兩個指令是否有用，然後再試一次 `insmod`
``` bash
sudo modprobe lz4
sudo modprobe lz4hc_compress
```
若要重新編譯、insert
```bash
sudo rmmod myf2fs
make clean
make && sudo insmod myf2fs.ko
```

# mount
```bash
# 從系統中獲取 CXL Region 的實體位址（Resource Address）與精確大小
sudo cxl list -R -u
# output 如下
# "region":"region13",
# "resource":"0x7840000000",
# "size":"33.00 GiB (35.43 GB)",    # 35433480192 bytes
# "type":"ram",
# "interleave_ways":1,
# "interleave_granularity":256,
# "decode_state":"commit"

# 建立掛載點
sudo mkdir /mnt/myf2fs_dax

sudo mount -t myf2fs -o "cxl_phys=0x7880000000,cxl_size=34359738368" /dev/dax13.0 /mnt/myf2fs_dax/
```



https://gemini.google.com/share/d0daf628c4ff