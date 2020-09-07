---
title: u-boot环境变量详解
type: uboot
description: 本文讲解了解uboot中环境变量的值首先采用flash ENV分区中的存储的值，如果分区不存在或者读取失败，那么会采用全局数组default_environment中的值作为默认环境变量
date: 2020-08-21 14:44:00
---

uboot中使用`printenv`命令查看环境变量（env），实际上是查看的内存中的值，这个内存使用 数据结构`env_t`来描述。只有当你执行`saveenv`
时，才会把内存中的env写入到flash的env分区中（如果你划分了ENV分区的话）。`saveenv`这个动作需要你代码（或手动）显示执行，才能把内存中的环境变量保存到flash分区中。

本文讲解了解uboot中环境变量的值首先采用flash ENV分区中的存储的值，如果分区不存在或者读取失败，那么会采用全局数组default_environment中的值作为默认环境变量。

## 平台信息

* u-boot-2016
* QCA-openwrt SDK
* ARMv7
* ipq6018
* 16MBnor + 128MBnand

## 分区信息

```
[    1.031260] 0x000000000000-0x0000000c0000 : "0:SBL1"
[    1.037285] 0x0000000c0000-0x0000000d0000 : "0:MIBIB"
[    1.042423] 0x0000000d0000-0x0000000f0000 : "0:BOOTCONFIG"
[    1.047402] 0x0000000f0000-0x000000110000 : "0:BOOTCONFIG1"
[    1.052780] 0x000000110000-0x0000002b0000 : "0:QSEE"
[    1.058250] 0x0000002b0000-0x000000450000 : "0:QSEE_1"
[    1.063360] 0x000000450000-0x000000460000 : "0:DEVCFG"
[    1.068339] 0x000000460000-0x000000470000 : "0:DEVCFG_1"
[    1.073463] 0x000000470000-0x0000004b0000 : "0:RPM"
[    1.078972] 0x0000004b0000-0x0000004f0000 : "0:RPM_1"
[    1.083561] 0x0000004f0000-0x000000500000 : "0:CDT"
[    1.088800] 0x000000500000-0x000000510000 : "0:CDT_1"
[    1.093474] 0x000000510000-0x000000520000 : "0:APPSBLENV"
[    1.098743] 0x000000520000-0x0000005c0000 : "0:APPSBL"
[    1.104027] 0x0000005c0000-0x000000660000 : "0:APPSBL_1"
[    1.109080] 0x000000660000-0x0000006a0000 : "0:ART"

```

本文环境平台flash分区中`0:APPSBLENV`分区用来存储环境变量。`APPSBL`为uboot分区。该分区是划分在nor flash上的。

## 代码解读

### common/board_f.c

```
static init_fnc_t init_sequence_f[] = {
    ...
    env_init,		/* initialize environment */
    ...
}
```

board/qca/arm/common/env.c

```
/*
 * Function description: Board specific initialization.
 * I/P : None
 * O/P : integer, 0 - no error.
 */
int env_init(void)
{
	int ret = 0;
	qca_smem_flash_info_t sfi;
    /*获取启动flash的类型*/
	smem_get_boot_flash(&sfi.flash_type,
			    &sfi.flash_index,
			    &sfi.flash_chip_select,
			    &sfi.flash_block_size,
			    &sfi.flash_density);

	if (sfi.flash_type == SMEM_BOOT_SPI_FLASH) {
#ifdef CONFIG_ENV_IS_IN_SPI_FLASH
    /*我们这里是SPI FLASH启动，所以走这个流程*/
		ret = sf_env_init();
#endif
#ifdef CONFIG_QCA_MMC
	} else if (sfi.flash_type == SMEM_BOOT_MMC_FLASH) {
		ret = mmc_env_init();
#endif
	} else {
		ret = nand_env_init();
	}

	return ret;
}
```

common/env_sf.c

```
int sf_env_init(void)
{
	/* SPI flash isn't usable before relocation */
	gd->env_addr = (ulong)&default_environment[0];  /*env_addr 指向默认环境变量数组，但是在本文环境中，这个值并没有用到*/
	gd->env_valid = 1;          /*这里赋值1， 会影响后面的流程*/

	return 0;
}
```

### common/board_r.c

```
init_fnc_t init_sequence_r[] = {
    ...
    initr_env,
    ...
}
```

common/board_r.c

```
static int initr_env(void)
{
	/* initialize environment */
	if (should_load_env())  /*这里会返回1*/
		env_relocate(); /*该分支会被执行*/
	else
		set_default_env(NULL);
#ifdef CONFIG_OF_CONTROL
	setenv_addr("fdtcontroladdr", gd->fdt_blob);
#endif

	/* Initialize from environment */
	load_addr = getenv_ulong("loadaddr", 16, load_addr);
#if defined(CONFIG_SYS_EXTBDINFO)
#if defined(CONFIG_405GP) || defined(CONFIG_405EP)
#if defined(CONFIG_I2CFAST)
	/*
	 * set bi_iic_fast for linux taking environment variable
	 * "i2cfast" into account
	 */
	{
		char *s = getenv("i2cfast");

		if (s && ((*s == 'y') || (*s == 'Y'))) {
			gd->bd->bi_iic_fast[0] = 1;
			gd->bd->bi_iic_fast[1] = 1;
		}
	}
#endif /* CONFIG_I2CFAST */
#endif /* CONFIG_405GP, CONFIG_405EP */
#endif /* CONFIG_SYS_EXTBDINFO */
	return 0;
}
```

```
/*
 * Tell if it's OK to load the environment early in boot.
 *
 * If CONFIG_OF_CONFIG is defined, we'll check with the FDT to see
 * if this is OK (defaulting to saying it's OK).
 *
 * NOTE: Loading the environment early can be a bad idea if security is
 *       important, since no verification is done on the environment.
 *
 * @return 0 if environment should not be loaded, !=0 if it is ok to load
 */
static int should_load_env(void)
{
#ifdef CONFIG_OF_CONTROL
#pragma message("CONFIG_OF_CONTROL marco is define")
    /*这里是在设备树中查找是否定义了 'load-environment' 这个节点，如果没有定义，该函数还是会返回第二个参数 1
    因为本文环境设备树中没有定义该节点，所以该函数会返回1 
        另外，根据上面对本函数的说明：‘过早的加载环境变量不是一个好的主意’

        所以在initr_env中，if分支的 env_relocate();会被执行
    */
	return fdtdec_get_config_int(gd->fdt_blob, "load-environment", 1);
#elif defined CONFIG_DELAY_ENVIRONMENT
#pragma message("CONFIG_DELAY_ENVIRONMENT marco is define")
	return 0;
#else
	return 1;
#endif
}
```
common/env_common.c

```
void env_relocate(void)
{
#if defined(CONFIG_NEEDS_MANUAL_RELOC)
	env_reloc();    /*这里描述了另一个给uboo添加cmd的方法，这里添加的是 env命令的选项部分。前提是上面那个宏开关要打开*/
	env_htab.change_ok += gd->reloc_off;
#endif
	if (gd->env_valid == 0) {   /*因为之前 已经赋值为1 */
#if defined(CONFIG_ENV_IS_NOWHERE) || defined(CONFIG_SPL_BUILD)
		/* Environment not changable */
		set_default_env(NULL);
#else
		bootstage_error(BOOTSTAGE_ID_NET_CHECKSUM);
		set_default_env("!bad CRC");
#endif
	} else {    /*走这个流程*/
		env_relocate_spec();    /*该函数是硬件相关函数，所以不同的平台的实现可能不一样*/
	}
}
```

board/qca/arm/common/env.c

```
void env_relocate_spec(void)
{
  printf("qca/arm/common/env(env_relocate_spec) is called\n");
	qca_smem_flash_info_t sfi;

    /*还是先判断 启动flash的类型*/
	smem_get_boot_flash(&sfi.flash_type,
			    &sfi.flash_index,
			    &sfi.flash_chip_select,
			    &sfi.flash_block_size,
			    &sfi.flash_density);

	if (sfi.flash_type == SMEM_BOOT_NO_FLASH) {
		set_default_env("!flashless boot");
#ifdef CONFIG_ENV_IS_IN_SPI_FLASH
	} else if (sfi.flash_type == SMEM_BOOT_SPI_FLASH) {
        /*这里会走这个spi 接口nor flash的分支*/
		sf_env_relocate_spec();
#endif
#ifdef CONFIG_QCA_MMC
	} else if (sfi.flash_type == SMEM_BOOT_MMC_FLASH) {
                mmc_env_relocate_spec();
#endif
	} else {
		nand_env_relocate_spec();
	}

};
```

common/env_sf.c

```
/*这两个是我添加的，用来在编译时查看宏值的*/
#define __PRINT_MACRO(x)  #x
#define PRINT_MACRO(x)  #x"="__PRINT_MACRO(x)
void sf_env_relocate_spec(void)
{
	int ret;
	char *buf = NULL;

    /*申请一段内存，内存大小根据这个宏来确定，其实就是0:APPSBLENV分区的大小*/
	buf = (char *)memalign(ARCH_DMA_MINALIGN, CONFIG_ENV_SIZE);

    /*spi flash读写固定格式，先probe一下，才能read、erase、write*/
	env_flash = spi_flash_probe(CONFIG_SF_DEFAULT_BUS, CONFIG_SF_DEFAULT_CS,
			CONFIG_SF_DEFAULT_SPEED, CONFIG_SF_DEFAULT_MODE);
	if (!env_flash) {
        /*如果probe失败，就采用默认的env*/
		set_default_env("!spi_flash_probe() failed");
		if (buf)
			free(buf);
		return;
	}
#pragma message(PRINT_MACRO(CONFIG_ENV_OFFSET))
#pragma message(PRINT_MACRO(CONFIG_ENV_RANGE))
  printf("CONFIG_ENV_OFFSET=%lld\n",CONFIG_ENV_OFFSET);
  printf("CONFIG_ENV_RANGE=%lld\n",CONFIG_ENV_RANGE);

    /*读取0:APPSBLENV 分区的内容*/
	ret = spi_flash_read(env_flash,
		CONFIG_ENV_OFFSET, CONFIG_ENV_RANGE, buf);
	if (ret) {
		set_default_env("!spi_flash_read() failed");
		goto out;
	}
  printf("sf_env_relocate_spec start import env.\n");
  print_buffer((ulong)buf, buf, 1, 1024, 16);
    /*将分区的内容，导入到我们使用 printenv 查看的内存中，其实是一个hash表*/
	ret = env_import(buf, 1);
	if (ret)
		gd->env_valid = 1;
out:
	if (buf)
		free(buf);
	env_flash = NULL;
}
```

common/env_common.c

```
/*
 * Check if CRC is valid and (if yes) import the environment.
 * Note that "buf" may or may not be aligned.
 */
int env_import(const char *buf, int check)
{
	env_t *ep = (env_t *)buf;
	int ret;

    /*crc校验*/
	if (check) {
		uint32_t crc;

		memcpy(&crc, &ep->crc, sizeof(crc));

        /*校验失败，使用默认数组中的环境变量*/
		if (crc32(0, ep->data, ENV_SIZE) != crc) {
			set_default_env("!bad CRC");
			return 0;
		}
	}

	/* Decrypt the env if desired. */
	ret = env_aes_cbc_crypt(ep, 0);
	if (ret) {
		error("Failed to decrypt env!\n");
		set_default_env("!import failed");
		return ret;
	}

    /*把从0:APPSBLENV分区读取的内容，导入到hash数组中，这些数据一直是在内存中的*/
	if (himport_r(&env_htab, (char *)ep->data, ENV_SIZE, '\0', 0, 0,
			0, NULL)) {
		gd->flags |= GD_FLG_ENV_READY;
		return 1;
	}

	error("Cannot import environment: errno = %d\n", errno);

	set_default_env("!import failed");

	return 0;
}
```

### 0:APPSBLENV 分区信息获取过程

board/qca/arm/common/board_init.c

```
int board_init(void)
{
	int ret;
	uint32_t start_blocks;
	uint32_t size_blocks;

    ....

	qgic_init();

	qca_smem_flash_info_t *sfi = &qca_smem_flash_info;

	gd->bd->bi_boot_params = QCA_BOOT_PARAMS_ADDR;
	gd->bd->bi_arch_number = smem_get_board_platform_type();

	ret = smem_get_boot_flash(&sfi->flash_type,
				  &sfi->flash_index,
				  &sfi->flash_chip_select,
				  &sfi->flash_block_size,
				  &sfi->flash_density);

    ....

    /*这里根据函数 smem_getpart 来在nor flash中查找 0:APPSBLENV 分区的起始地址 和 分区大小*/
	if ((sfi->flash_type != SMEM_BOOT_MMC_FLASH) && (sfi->flash_type != SMEM_BOOT_NO_FLASH))  {
		ret = smem_getpart("0:APPSBLENV", &start_blocks, &size_blocks);
		if (ret < 0) {
			printf("cdp: get environment part failed\n");
			return ret;
		}

		board_env_offset = ((loff_t) sfi->flash_block_size) * start_blocks;
		board_env_size = ((loff_t) sfi->flash_block_size) * size_blocks;
		
    printf("board_env_offset=%lld\n",board_env_offset);
    printf("board_env_size=%lld\n",board_env_size);
	}

    /*根据flash类型不同，初始化全局变量 board_env_range*/
	switch (sfi->flash_type) {
	case SMEM_BOOT_NAND_FLASH:
	case SMEM_BOOT_QSPI_NAND_FLASH:
		board_env_range = CONFIG_ENV_SIZE_MAX;
		BUG_ON(board_env_size < CONFIG_ENV_SIZE_MAX);
		break;
	case SMEM_BOOT_SPI_FLASH:
		board_env_range = board_env_size;
		BUG_ON(board_env_size > CONFIG_ENV_SIZE_MAX);
		break;
#ifdef CONFIG_QCA_MMC
	case SMEM_BOOT_MMC_FLASH:
	case SMEM_BOOT_NO_FLASH:
		board_env_range = CONFIG_ENV_SIZE_MAX;
		break;
#endif
	default:
		printf("BUG: unsupported flash type : %d\n", sfi->flash_type);
		BUG();
	}

    /*根据不同flash类型，初始化saveenv 这个函数指针*/
	if (sfi->flash_type == SMEM_BOOT_SPI_FLASH) {
#ifdef CONFIG_ENV_IS_IN_SPI_FLASH
		saveenv = sf_saveenv;
		env_name_spec = sf_env_name_spec;
#endif
#ifdef CONFIG_QCA_MMC
	} else if (sfi->flash_type == SMEM_BOOT_MMC_FLASH) {
		saveenv = mmc_saveenv;
		env_ptr = mmc_env_ptr;
		env_name_spec = mmc_env_name_spec;
#endif
	} else {
		saveenv = nand_saveenv;
		env_ptr = nand_env_ptr;
		env_name_spec = nand_env_name_spec;
	}
#endif
	ret = ipq_board_usb_init();
	if (ret < 0) {
		printf("WARN: ipq_board_usb_init failed\n");
	}

	aquantia_phy_reset_init();
	disable_audio_clks();
	ipq_uboot_fdt_fixup();
	/*
	 * Needed by ipq806x to avoid TX FIFO curruption during
	 * serial init after relocation
	 */
	uart_wait_tx_empty();
	return 0;
}
```
