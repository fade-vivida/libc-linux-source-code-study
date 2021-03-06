# Glibc文件流函数的实现机制 #
源代码库版本为：glibc2.27

glibc中关于文件流函数（fopen，fread，fwrite等）的实现源代码位于libio目录下。  
# 1. fopen #
**iofopen.c** 是fopen函数实现的关键代码，其中包含以下两个关键函数。
<pre class="prettyprint lang-javascript"> 
_IO_FILE *
_IO_new_fopen (const char *filename, const char *mode)
{
  	return __fopen_internal (filename, mode, 1);
}
</pre>
当我们平时在写C程序调用fopen函数时，实际调用的函数就是\_IO\_new\_fopen函数（在iofopen.c中可以看到关于fopen与该函数的链接及符号对应）。然后该函数又会调用\_\_fopen\_internal实现真正的函数功能。

## 1.1 \_\_fopen\_internal函数关键代码如下所示 ##
<pre class="prettyprint lang-javascript"> 
_IO_FILE * __fopen_internal (const char *filename, const char *mode, int is32)
{
	struct locked_FILE
	{
		struct _IO_FILE_plus fp;
		#ifdef _IO_MTSAFE_IO
			_IO_lock_t lock;
		#endif
	struct _IO_wide_data wd;
	} *new_f = (struct locked_FILE *) malloc (sizeof (struct locked_FILE));
	//在该函数中又定义了一个结构体，该结构体其实主要包含3个元素，(struct _IO_FILE_plus)fp，_IO_lock_t lock，(struct _IO_wide_data)wd
	if (new_f == NULL)
		return NULL;
	#ifdef _IO_MTSAFE_IO
		new_f->fp.file._lock = &new_f->lock;
	#endif
<a href = "#1">	_IO_no_init (&new_f->fp.file, 0, 0, &new_f->wd, &_IO_wfile_jumps);	//调用_IO_old_init函数进行初始化</a>
	_IO_JUMPS (&new_f->fp) = &_IO_file_jumps;	//设置fp的vtable字段，虚函数表
<a href = "#2">	_IO_new_file_init_internal (&new_f->fp);	//再次调用初始化函数（该函数完成其他的初始化功能）</a>
	#if  !_IO_UNIFIED_JUMPTABLES
		new_f->fp.vtable = NULL;
	#endif
	if (_IO_file_fopen ((_IO_FILE *) new_f, filename, mode, is32) != NULL)		//该函数在源代码中无法找到定义，目前悬而未解。。。。
		return __fopen_maybe_mmap (&new_f->fp.file);
<a href = "#3">	_IO_un_link (&new_f->fp);	//从_IO_list_all链表上拆下刚new_f->fp</a>
	//如果程序运行到这里，说明打开文件失败，此时将文件指针从链表上拆下，并释放该堆块。
	free (new_f);
	return NULL;
}
</pre>
从源代码中可以看出，该函数功能为创建一个locked\_FILE类型的结构体new\_f（该结构体包含3个元素fp，lock，wd）,然后对new\_f进行一系列初始化（包括初始化\_IO\_FILE\_plus结构体各个字段，将fp加入\_IO\_list\_all链表等），最后调用\_IO\_file\_fopen函数打开该文件流指针。
<a name = "1"></a>
## 1.2 \_IO\_no\_init函数实现源代码 ##
<pre class="prettyprint lang-javascript"> 
void _IO_no_init (_IO_FILE *fp, int flags, int orientation,struct _IO_wide_data *wd, const struct _IO_jump_t *jmp)
{
	_IO_old_init (fp, flags);
	fp->_mode = orientation;	//初始化时，fp->_mode = 0
	if (orientation >= 0)
	{
		//对（struct _IO_FILE)fp的_wide_data字段进行初始化
		fp->_wide_data = wd;
		fp->_wide_data->_IO_buf_base = NULL;
		fp->_wide_data->_IO_buf_end = NULL;
		fp->_wide_data->_IO_read_base = NULL;
		fp->_wide_data->_IO_read_ptr = NULL;
		fp->_wide_data->_IO_read_end = NULL;
		fp->_wide_data->_IO_write_base = NULL;
		fp->_wide_data->_IO_write_ptr = NULL;
		fp->_wide_data->_IO_write_end = NULL;
		fp->_wide_data->_IO_save_base = NULL;
		fp->_wide_data->_IO_backup_base = NULL;
		fp->_wide_data->_IO_save_end = NULL;
		
		fp->_wide_data->_wide_vtable = jmp;
	}
	else
		/* Cause predictable crash when a wide function is called on a byte stream.  */
		fp->_wide_data = (struct _IO_wide_data *) -1L;
	fp->_freeres_list = NULL;
}
</pre>
## 1.3 \_IO\_old\_init函数实现代码 ##
<pre class="prettyprint lang-javascript"> 
void _IO_old_init (_IO_FILE *fp, int flags)
{
	fp->_flags = _IO_MAGIC|flags;	//_IO_MAGIC = 0xFBAD0000 魔数
	fp->_flags2 = 0;
	if (stdio_needs_locking)
	fp->_flags2 |= _IO_FLAGS2_NEED_LOCK;	//_IO_FLAGS2_NEED_LOCK = 128
	fp->_IO_buf_base = NULL;
	fp->_IO_buf_end = NULL;
	fp->_IO_read_base = NULL;
	fp->_IO_read_ptr = NULL;
	fp->_IO_read_end = NULL;
	fp->_IO_write_base = NULL;
	fp->_IO_write_ptr = NULL;
	fp->_IO_write_end = NULL;
	fp->_chain = NULL; /* Not necessary. */
	
	fp->_IO_save_base = NULL;
	fp->_IO_backup_base = NULL;
	fp->_IO_save_end = NULL;
	fp->_markers = NULL;
	fp->_cur_column = 0;
	#if _IO_JUMPS_OFFSET
		fp->_vtable_offset = 0;
	#endif
	#ifdef _IO_MTSAFE_IO
	  if (fp->_lock != NULL)
		_IO_lock_init (*fp->_lock);
	#endif
}
</pre>
<a name="2"></a>
## 1.4 \_IO\_new\_file\_init\_internal函数实现代码 ##

<pre class="prettyprint lang-javascript"> 
void _IO_new_file_init_internal (struct _IO_FILE_plus *fp)
{
	/* POSIX.1 allows another file handle to be used to change the position of our file descriptor.
	Hence we actually don't know the actual position before we do the first fseek (and until a
	following fflush). */
	fp->file._offset = _IO_pos_BAD;		//_IO_pos_BAD = -1
	fp->file._IO_file_flags |= CLOSED_FILEBUF_FLAGS;		
<a href = "#4">	_IO_link_in (fp);	//将fp插入_IO_list_all链表中，插入位置为链表头部</a>
	fp->file._fileno = -1;
}
</pre>
CLOSED\_FILEBUF\_FLAGS定义如下：
<pre class="prettyprint lang-javascript"> 
#define CLOSED_FILEBUF_FLAGS \
	(_IO_IS_FILEBUF+_IO_NO_READS+_IO_NO_WRITES+_IO_TIED_PUT_GET)	
	//_IO_IS_FILEBUF = 0x2000, _IO_NO_READ = 0x4, _IO_NO_WRITES = 0x8, _IO_TIED_PUT_GET = 0x400
</pre>
可以看到\_IO\_NO\_WRITES = 0x8,\_IO\_NO\_READ = 0x4。以stdin和stdout为例，stdin为标准输入，stdout为标准输出，理论上stdin文件流标识符中应该包含\_IO\_NO\_READ，stdout文件流标识符中应该包含\_IO\_NO\_WRITES。  

实际情况如下所示：  
![test](https://raw.githubusercontent.com/fade-vivida/libc-linux-source-code-study/master/libc_study/picture/io1.JPG)  
实际结果与猜想一致。

<a name="3"></a>
## 1.5 \_IO\_un\_link函数实现代码 ##

该函数功能为从\_IO\_list\_all链表上拆下fp指针。
<pre class="prettyprint lang-javascript"> 
_IO_un_link (struct _IO_FILE_plus *fp)
{
	if (fp->file._flags & _IO_LINKED)	//_IO_LINKED = 0x80
	{
		struct _IO_FILE **f;
		#ifdef _IO_MTSAFE_IO
			_IO_cleanup_region_start_noarg (flush_cleanup);
			_IO_lock_lock (list_all_lock);
			run_fp = (_IO_FILE *) fp;
			_IO_flockfile ((_IO_FILE *) fp);
		#endif
		if (_IO_list_all == NULL);
		else if (fp == _IO_list_all)
			_IO_list_all = (struct _IO_FILE_plus *) _IO_list_all->file._chain;
		else
			for (f = &_IO_list_all->file._chain; *f; f = &(*f)->_chain)
				if (*f == (_IO_FILE *) fp)
				{
					*f = fp->file._chain;
					break;
				}
		fp->file._flags &= ~_IO_LINKED;
		#ifdef _IO_MTSAFE_IO
			_IO_funlockfile ((_IO_FILE *) fp);
			run_fp = NULL;
			_IO_lock_unlock (list_all_lock);
			_IO_cleanup_region_end (0);
		#endif
	}
}
</pre>
<a name = "4"></a>
## 1.6 \_IO\_link\_in函数实现代码 ##

该函数功能为将fp标志的文件流指针插入\_IO\_list\_all链表中。
<pre class="prettyprint lang-javascript"> 
void _IO_link_in (struct _IO_FILE_plus *fp)
{
	if ((fp->file._flags & _IO_LINKED) == 0)
	{
		fp->file._flags |= _IO_LINKED;
		#ifdef _IO_MTSAFE_IO
			_IO_cleanup_region_start_noarg (flush_cleanup);
			_IO_lock_lock (list_all_lock);
			run_fp = (_IO_FILE *) fp;
			_IO_flockfile ((_IO_FILE *) fp);
		#endif
      	
		fp->file._chain = (_IO_FILE *) _IO_list_all;	
		_IO_list_all = fp;
		//将fp插入到链表头部
		
		#ifdef _IO_MTSAFE_IO
			_IO_funlockfile ((_IO_FILE *) fp);
			run_fp = NULL;
			_IO_lock_unlock (list_all_lock);
			_IO_cleanup_region_end (0);
		#endif
	}
}
</pre>
# 2. fread #
**注：通过下面的分析可以知道其实fread函数在实现功能时，真正调用的时函数列表中的\_\_xsgetn函数**
<pre class="prettyprint lang-javascript"> 
_IO_size_t _IO_fread (void *buf, _IO_size_t size, _IO_size_t count, _IO_FILE *fp)
{
	_IO_size_t bytes_requested = size * count;	//读入的总字节数
  	_IO_size_t bytes_read;
  	CHECK_FILE (fp, 0);		//对文件指针进行检查
  	if (bytes_requested == 0)
    	return 0;
  	_IO_acquire_lock (fp);
<a href = "#5">	bytes_read = _IO_sgetn (fp, (char *) buf, bytes_requested);	//调用_IO_sgetn 函数从fp文流中读入bytes_requested字节的数据到buf中</a>
  	_IO_release_lock (fp);
  	return bytes_requested == bytes_read ? count : bytes_read / size;
}
</pre>
函数的返回值为实际读入的数据项的个数（之所以这么说是因为fread函数的第二个参数为读入数据类型，第三个参数为该类型数据的个数）。
## 2.1 CHECK\_FILE宏 ##
该宏定义的功能为检查fp指针是否合法，如果当前为调试模式，则检查fp是否为NULL。若为NULL，则返回0。否则，检查fp指针的\_IO\_file\_flags字段的高word是否为\_IO\_MAGIC(0xfbad)。
<pre class="prettyprint lang-javascript"> 
#ifdef IO_DEBUG
# define CHECK_FILE(FILE, RET) \
	if ((FILE) == NULL) 
	{ 
		MAYBE_SET_EINVAL; 
		return RET; 
	}
	else 
	{ 
		COERCE_FILE(FILE); 
	    if (((FILE)->_IO_file_flags & _IO_MAGIC_MASK) != _IO_MAGIC)		//_IO_MAGIC_MASK = 0xffff0000,_IO_MAGIC = 0xfbad0000
	  	{ 
			MAYBE_SET_EINVAL; 
			return RET; 
		}
	}
#else
	#define CHECK_FILE(FILE, RET) COERCE_FILE (FILE)
#endif

# define COERCE_FILE(FILE) /* Nothing */
</pre>

<a name = "5"></a>
## 2.2 \_IO\_sgetn函数 ##

可以看到该函数只是单纯的调用了\_IO\_XSGETN宏。
<pre class="prettyprint lang-javascript"> 
_IO_size_t
_IO_sgetn (_IO_FILE *fp, void *data, _IO_size_t n)
{
	/* FIXME handle putback buffer here! */
	return _IO_XSGETN (fp, data, n);
}
</pre>
该宏及相关宏定义代码如下所示：  
	
	#define _IO_XSGETN(FP, DATA, N) JUMP2 (__xsgetn, FP, DATA, N)	//实际调用函数为__xsgetn
	#define JUMP2(FUNC, THIS, X1, X2) (_IO_JUMPS_FUNC(THIS)->FUNC) (THIS, X1, X2)
该宏再次调用了JUMP2宏，可以看出JUMP2宏的功能为调用一个函数列表中的函数（函数有两个参数）。  

在调用函数前还会对该函数指针列表的合法性进行检查，相关代码如下所示：
<pre class="prettyprint lang-javascript"> 
#if _IO_JUMPS_OFFSET	//如果使用了老版本的的_IO_FILE结构体，_IO_JUMPS_OFFSET为1，否则为0
	# define _IO_JUMPS_FUNC(THIS) \
		(IO_validate_vtable                                                   \
		(*(struct _IO_jump_t **) ((void *) &_IO_JUMPS_FILE_plus (THIS) + (THIS)->_vtable_offset)))	
		//也就是说如果是老版本，还修改vtable后，还需要将_vtable_offset字段置0
	# define _IO_vtable_offset(THIS) (THIS)->_vtable_offset
#else
	# define _IO_JUMPS_FUNC(THIS) (IO_validate_vtable (_IO_JUMPS_FILE_plus (THIS)))
	# define _IO_vtable_offset(THIS) 0
#endif

#define _IO_JUMPS_FILE_plus(THIS) _IO_CAST_FIELD_ACCESS ((THIS), struct _IO_FILE_plus, vtable)
/* Type of MEMBER in struct type TYPE.  */
#define _IO_MEMBER_TYPE(TYPE, MEMBER) __typeof__ (((TYPE){}).MEMBER)
//返回成员变量的类型	

/* Essentially ((TYPE *) THIS)->MEMBER, but avoiding the aliasing violation in case THIS has a different pointer type.  */
#define _IO_CAST_FIELD_ACCESS(THIS, TYPE, MEMBER) \
(*(_IO_MEMBER_TYPE (TYPE, MEMBER) *)(((char *) (THIS)) + offsetof(TYPE, MEMBER)))
//返回THIS所标志的结构体中成员MEMBER的地址
</pre>
进一步分析其调用的宏定义，可以看出程序首先通过固定偏移找到vtable成员变量，然后调用IO\_validata\_vtable函数检查该vtable变量的合法性。如果vtable函数指针列表合法，则调用该函数（\_\_xsgetn)。  

\_IO\_file\_plus结构体的vtable字段示意图：  
![vtable](https://raw.githubusercontent.com/fade-vivida/libc-linux-source-code-study/master/libc_study/picture/io_jump.JPG)

<a name = "6"></a>
## 2.3 vtable指针合法性检查 ##

IO\_validata\_vtable函数代码如下所示：
<pre class="prettyprint lang-javascript"> 
static inline const struct _IO_jump_t *
IO_validate_vtable (const struct _IO_jump_t *vtable)
{
	/* Fast path: The vtable pointer is within the __libc_IO_vtables section.  */
	uintptr_t section_length = __stop___libc_IO_vtables - __start___libc_IO_vtables;
	const char *ptr = (const char *) vtable;
	uintptr_t offset = ptr - __start___libc_IO_vtables;
	if (__glibc_unlikely (offset >= section_length))
		/* The vtable pointer is not in the expected section.  Use the slow path, which will terminate the process if necessary.  */
		_IO_vtable_check ();
	return vtable;
}
</pre>
glibc默认的文件流函数指针调用列表位于一个固定的节中，因此一个快速的检查方法就是看vtable是否在该区间内，如果不是就必须采用较慢的更为细致的检测方法（\_IO\_vtable\_check)。  

\_IO\_vtable\_check函数代码如下：
<pre class="prettyprint lang-javascript"> 
void attribute_hidden _IO_vtable_check (void)
{
	#ifdef SHARED
		/* Honor the compatibility flag.  */
		void (*flag) (void) = atomic_load_relaxed (&IO_accept_foreign_vtables);
	#ifdef PTR_DEMANGLE
		PTR_DEMANGLE (flag);
	#endif
	if (flag == &_IO_vtable_check)
		return;
	
	/* In case this libc copy is in a non-default namespace, we always
	need to accept foreign vtables because there is always a
	possibility that FILE * objects are passed across the linking
	boundary.  */
	{
		Dl_info di;
		struct link_map *l;
		if (!rtld_active ()
			|| (_dl_addr (_IO_vtable_check, &di, &l, NULL) != 0
			&& l->l_ns != LM_ID_BASE))
			return;
	}
	
	#else /* !SHARED */
	/* We cannot perform vtable validation in the static dlopen case
	because FILE * handles might be passed back and forth across the
	boundary.  Therefore, we disable checking in this case.  */
	if (__dlopen != NULL)
		return;
	#endif
	__libc_fatal ("Fatal error: glibc detected an invalid stdio handle\n");
}
</pre>
该函数看不懂，大体意思应该是定义了共享模式和非共享模式下文件流vtable字段的检测方法。从实际应用中发现，在libc2.24之前可以在堆中放置一个vtable函数指针列表（伪造\_IO\_FILE结构），但在libc2.24之后该方法不可用，还有新的利用方法，未完待续。。。。见下文

## 2.4 \_IO\_file\_xsgetn函数 ##
<pre class = "prettyprint lang-javascript">
_IO_size_t
_IO_file_xsgetn (_IO_FILE *fp, void *data, _IO_size_t n)
{
	_IO_size_t want, have;
	_IO_ssize_t count;
	char *s = data;
	
	want = n;
	
	if (fp->_IO_buf_base == NULL)
	{
		//如果文件流缓冲区为null
		/* Maybe we already have a push back pointer.  */
		if (fp->_IO_save_base != NULL)
		{
			free (fp->_IO_save_base);
			fp->_flags &= ~_IO_IN_BACKUP;
		}
		<a href = "#7">_IO_doallocbuf (fp);		//这里会调用vtable->doallocate</a>
	}
	
	while (want > 0)
	{
		have = fp->_IO_read_end - fp->_IO_read_ptr;
		if (want <= have)
		{
			memcpy (s, fp->_IO_read_ptr, want);
			fp->_IO_read_ptr += want;
			want = 0;
			//当前流缓冲的数据大于请求数据
		}
		else
		{
			if (have > 0)
			{
				#ifdef _LIBC
					s = __mempcpy (s, fp->_IO_read_ptr, have);
				#else
					memcpy (s, fp->_IO_read_ptr, have);
					s += have;
				#endif
				want -= have;
				fp->_IO_read_ptr += have;
			}
		
			/* Check for backup and repeat */
			if (_IO_in_backup (fp))
			{
				//如果当前文件流设置有备份标志位0x100，则交换
					_IO_read_base<-->_IO_save_base
					_IO_read_end<-->_IO_save_end
				<a href = "#8">_IO_switch_to_main_get_area (fp);</a>
				continue;
			}
			
			/* If we now want less than a buffer, underflow and repeat
			the copy.  Otherwise, _IO_SYSREAD directly to
			the user buffer. */
			
			if (fp->_IO_buf_base
				&& want < (size_t) (fp->_IO_buf_end - fp->_IO_buf_base))
			{
				//关键点：如果当前缓冲区大小大于请求字节数（fp->_IO_buf_end - fp->_IO_buf_base > want)，
				//那么就先调用underflow函数读入（buff size大小的数据）到缓冲区中，然后再次循环后，
				//使用memcpy函数复制实际请求大小。
				if (<a href = "#9">__underflow (fp)</a> == EOF)
					break;
				
				continue;
			}
			
			/* These must be set before the sysread as we might longjmp out 
			waiting for input. */
			_IO_setg (fp, fp->_IO_buf_base, fp->_IO_buf_base, fp->_IO_buf_base);
			_IO_setp (fp, fp->_IO_buf_base, fp->_IO_buf_base);
			
			/* Try to maintain alignment: read a whole number of blocks.  */
			count = want;
			
			if (fp->_IO_buf_base)
			{
				_IO_size_t block_size = fp->_IO_buf_end - fp->_IO_buf_base;
				if (block_size >= 128)
					count -= want % block_size;
			}
			//否则，直接将数据读入到目标buff中（考虑对齐）
			count = _IO_SYSREAD (fp, s, count);
			if (count <= 0)
			{
				if (count == 0)
					fp->_flags |= _IO_EOF_SEEN;
				else
					fp->_flags |= _IO_ERR_SEEN;
				break;
			}
			s += count;
			want -= count;
			if (fp->_offset != _IO_pos_BAD)
				_IO_pos_adjust (fp->_offset, count);
		}
	}
	return n - want;
}
libc_hidden_def (_IO_file_xsgetn)
</pre>
<a name = "8"></a>
## 2.5 \_IO\_switch\_to\_main\_get函数 ##
\_IO\_switch\_to\_main\_get\_area函数定义如下：
<pre class = "prettyprint lang-javascript">
void _IO_switch_to_main_get_area (_IO_FILE *fp)
{
	char *tmp;
	fp->_flags &= ~_IO_IN_BACKUP;
	
	/* Swap _IO_read_end and _IO_save_end. */
	tmp = fp->_IO_read_end;
	fp->_IO_read_end = fp->_IO_save_end;
	fp->_IO_save_end= tmp;

	/* Swap _IO_read_base and _IO_save_base. */
	tmp = fp->_IO_read_base;
	fp->_IO_read_base = fp->_IO_save_base;
	fp->_IO_save_base = tmp;

	/* Set _IO_read_ptr. */
	fp->_IO_read_ptr = fp->_IO_read_base;
}
</pre>
该函数的主要功能为：如果当前文件流标志位为备份缓存（fp->flag & _IO_IN_BACKUP == 0x100），则将\_IO\_read\_base，\_IO\_read\_ptr，\_IO\_read\_end与\_IO\_save\_base，\_IO\_save\_end进行交换。  

也就是说当前文件流有两个缓存区，一个是主缓存，一个是备份缓存，该标志位的作用就是表明当前是处于哪个缓存空间。
<a name = "9"></a>
## 2.6 \_IO\_new\_file\_underflow函数 ##
该函数为vtable->\_\_underflow hook的功能函数，其函数代码如下所示：
<pre class = "prettyprint lang-javascript">
int _IO_new_file_underflow (_IO_FILE *fp)
{
	_IO_ssize_t count;
	#if 0
	/* SysV does not make this test; take it out for compatibility */
	if (fp->_flags & _IO_EOF_SEEN)
		return (EOF);
	#endif
	if (fp->_flags & _IO_NO_READS)
	{
		//文件流标志为不可读，则直接返回EOF
		fp->_flags |= _IO_ERR_SEEN;
		__set_errno (EBADF);
		return EOF;
	}
	if (fp->_IO_read_ptr < fp->_IO_read_end)
		return *(unsigned char *) fp->_IO_read_ptr;		//如果fp->_IO_read_ptr < fp->_IO_read_end表示流缓存还有数据为读，不用调用SYS_READ。
	
	if (fp->_IO_buf_base == NULL)
	{
		/* Maybe we already have a push back pointer.  */
		if (fp->_IO_save_base != NULL)
		{
			free (fp->_IO_save_base);
			fp->_flags &= ~_IO_IN_BACKUP;
		}
		<a href = "#7">_IO_doallocbuf (fp)</a>;		//分配新的流缓存
	}
	
	/* Flush all line buffered files before reading. */
	/* FIXME This can/should be moved to genops ?? */
	if (fp->_flags & (_IO_LINE_BUF|_IO_UNBUFFERED))
	{
		//根据源代码注释，该段代码只是为了兼容之前Unix对stdout所做的操作（flush stdout），其他并无实际作用。
		#if 0
			_IO_flush_all_linebuffered ();
		#else
			/* We used to flush all line-buffered stream.  This really isn't
			required by any standard.  My recollection is that
			traditional Unix systems did this for stdout.  stderr better
			not be line buffered.  So we do just that here
			explicitly.  --drepper */
			
			IO_acquire_lock (_IO_stdout);
			
			if ((_IO_stdout->_flags & (_IO_LINKED | _IO_NO_WRITES | _IO_LINE_BUF))
				== (_IO_LINKED | _IO_LINE_BUF))
				_IO_OVERFLOW (_IO_stdout, EOF);
			_IO_release_lock (_IO_stdout);
		#endif
	}
	
	<a href = "#10">_IO_switch_to_get_mode (fp);</a>
	//对于可读可写的文件流，在读入数据前要先确保其写入内容都已完成写入

	/* This is very tricky. We have to adjust those
	pointers before we call _IO_SYSREAD () since
	we may longjump () out while waiting for
	input. Those pointers may be screwed up. H.J. */
	fp->_IO_read_base = fp->_IO_read_ptr = fp->_IO_buf_base;
	fp->_IO_read_end = fp->_IO_buf_base;
	fp->_IO_write_base = fp->_IO_write_ptr = fp->_IO_write_end = fp->_IO_buf_base;
	
	count = _IO_SYSREAD (fp, fp->_IO_buf_base,fp->_IO_buf_end - fp->_IO_buf_base);
	//向缓冲区buff中读入数据
	if (count <= 0)
	{
		if (count == 0)
			fp->_flags |= _IO_EOF_SEEN;
		else
			fp->_flags |= _IO_ERR_SEEN, count = 0;
	}
	fp->_IO_read_end += count;
	if (count == 0)
	{
		/* If a stream is read to EOF, the calling application may switch active
		handles.  As a result, our offset cache would no longer be valid, so
		unset it.  */
		fp->_offset = _IO_pos_BAD;
		return EOF;
	}
	if (fp->_offset != _IO_pos_BAD)
		_IO_pos_adjust (fp->_offset, count);
	return *(unsigned char *) fp->_IO_read_ptr;
}
libc_hidden_ver (_IO_new_file_underflow, _IO_file_underflow)
</pre>
从函数源码分析可知，该函数的主要功能为在read buff中的数据都被读完时（\_IO\_read\_ptr == \_IO\_read\_end），

<a name = "7"></a>
## 2.7 \_IO\_doallocbuf ##
<pre class = "prettyprint lang-javascript">
void
_IO_doallocbuf (_IO_FILE *fp)
{
	if (fp->_IO_buf_base)
		return;
	if (!(fp->_flags & _IO_UNBUFFERED) || fp->_mode > 0)
		if (_IO_DOALLOCATE (fp) != EOF)		//调用vtable函数列表中的__doallocate函数
			return;
	_IO_setb (fp, fp->_shortbuf, fp->_shortbuf+1, 0);
}
libc_hidden_def (_IO_doallocbuf)
</pre>
其中\_IO\_DOALLOCATE为调用vtable->\_\_doallocate，该函数为一个hook函数，其真实实现函数为\_IO\_file\_doallocate，该函数关键代码如下所示：
<pre class = "prettyprint lang-javascript">
/* Allocate a file buffer, or switch to unbuffered I/O.  Streams for
   TTY devices default to line buffered.  */
int
_IO_file_doallocate (_IO_FILE *fp)
{
	......
	p = malloc (size);
	if (__glibc_unlikely (p == NULL))
		return EOF;
	_IO_setb (fp, p, p + size, 1);
	return 1;
}

void _IO_setb (_IO_FILE *f, char *b, char *eb, int a)
{
	if (f->_IO_buf_base && !(f->_flags & _IO_USER_BUF))		//_IO_USER_BUF = 0x1
		//其中_IO_USER_BUF的定义为：用户自己的buff，在close是不能进行free
		free (f->_IO_buf_base);
	f->_IO_buf_base = b;
	f->_IO_buf_end = eb;
	if (a)
		f->_flags &= ~_IO_USER_BUF;
	else
		f->_flags |= _IO_USER_BUF;
}
libc_hidden_def (_IO_setb)
</pre>
可以看到该函数功能为：重新为fp文件流分配一个buff缓冲区。

<a name = "#10"></a>
## 2.8 \_IO\_switch\_to\_get\_mode函数 ##
该函数功能为：将当前文件流由写模式（put mode）转化为读模式（get mode）。
<pre class = "prettyprint lang-javascript">
int _IO_switch_to_get_mode (_IO_FILE *fp)
{
	if (fp->_IO_write_ptr > fp->_IO_write_base)
		if (<a href = "#9">_IO_OVERFLOW (fp, EOF)</a> == EOF)
			return EOF;
	if (_IO_in_backup (fp))
		fp->_IO_read_base = fp->_IO_backup_base;
	else
	{
		fp->_IO_read_base = fp->_IO_buf_base;
		if (fp->_IO_write_ptr > fp->_IO_read_end)
			fp->_IO_read_end = fp->_IO_write_ptr;
	}
	fp->_IO_read_ptr = fp->_IO_write_ptr;
	fp->_IO_write_base = fp->_IO_write_ptr = fp->_IO_write_end = fp->_IO_read_ptr;
	fp->_flags &= ~_IO_CURRENTLY_PUTTING;
	return 0;
}
libc_hidden_def (_IO_switch_to_get_mode)
</pre>
# 3. fwrite #
**注：最终调用vtable中的\_\_xsputn**
<pre class="prettyprint lang-javascript"> 
_IO_size_t _IO_fwrite (const void *buf, _IO_size_t size, _IO_size_t count, _IO_FILE *fp)
{
	_IO_size_t request = size * count;
	_IO_size_t written = 0;
	CHECK_FILE (fp, 0);
	if (request == 0)
		return 0;
	_IO_acquire_lock (fp);
	if (_IO_vtable_offset (fp) != 0 || _IO_fwide (fp, -1) == -1)
		written = _IO_sputn (fp, (const char *) buf, request);		//最终调用_IO_sputn
	_IO_release_lock (fp);
	/* We have written all of the input in case the return value indicates
	this or EOF is returned.  The latter is a special case where we
	simply did not manage to flush the buffer.  But the data is in the
	buffer and therefore written as far as fwrite is concerned.  */
	if (written == request || written == EOF)
		//这里需要注意written返回EOF是因为数据已经被写入到了内核缓冲中，但由于没有fflush，所以返回EOF
		return count;
	else
		return written / size;
}
</pre>
fwrite函数与fread函数逻辑相似，fwrite函数最终调用的vtable函数链表中的函数为\_\_xsputn。

## 3.1 \_IO\_new\_file\_xsputn函数 ##
<pre class = "prettyprint lang-javascript">
_IO_size_t _IO_new_file_xsputn (_IO_FILE *f, const void *data, _IO_size_t n)
{
	const char *s = (const char *) data;
	_IO_size_t to_do = n;
	int must_flush = 0;
	_IO_size_t count = 0;

	if (n <= 0)
		return 0;
	
	/* This is an optimized implementation.
	If the amount to be written straddles a block boundary
	(or the filebuf is unbuffered), use sys_write directly. */

	/* First figure out how much space is available in the buffer. */
	if ((f->_flags & _IO_LINE_BUF) && (f->_flags & _IO_CURRENTLY_PUTTING))
	{
		count = f->_IO_buf_end - f->_IO_write_ptr;
		if (count >= n)
		{
			const char *p;
			for (p = s + n; p > s; )
			{
				if (*--p == '\n')
				{
					count = p - s + 1;
					must_flush = 1;
					break;
				}
			}
		}
	}
	else if (f->_IO_write_end > f->_IO_write_ptr)
		count = f->_IO_write_end - f->_IO_write_ptr; /* Space available. */

	/* Then fill the buffer. */
	if (count > 0)
	{
		if (count > to_do)
			count = to_do;
		#ifdef _LIBC
			f->_IO_write_ptr = __mempcpy (f->_IO_write_ptr, s, count);
		#else
			memcpy (f->_IO_write_ptr, s, count);
		f->_IO_write_ptr += count;
		#endif
		s += count;
		to_do -= count;
	}
	if (to_do + must_flush > 0)
	{
		_IO_size_t block_size, do_write;
		/* Next flush the (full) buffer. */
		if (<a href = "#19">_IO_OVERFLOW (f, EOF)</a> == EOF)
		/* If nothing else has to be written we must not signal the caller that everything has been written.  */
			return to_do == 0 ? EOF : n - to_do;
		
		/* Try to maintain alignment: write a whole number of blocks.  */
		block_size = f->_IO_buf_end - f->_IO_buf_base;
		do_write = to_do - (block_size >= 128 ? to_do % block_size : 0);

		if (do_write)
		{
			count = new_do_write (f, s, do_write);
			to_do -= count;
			if (count < do_write)
				return n - to_do;
		}

		/* Now write out the remainder.  Normally, this will fit in the
		buffer, but it's somewhat messier for line-buffered files,
		so we let _IO_default_xsputn handle the general case. */
		if (to_do)
			to_do -= _IO_default_xsputn (f, s+do_write, to_do);
	}
	return n - to_do;
}
libc_hidden_ver (_IO_new_file_xsputn, _IO_file_xsputn)
</pre>

<a name = "19"></a>
## 3.2 \_IO\_new\_file\_overflow函数 ##
该函数功能为：对当前文件流的写缓存进行刷新。具体逻辑流程如下所示：  
1. 如果当前文件流不可写（\_IO\_NO\_WRITES 0x2），直接返回EOF  
2. 如果当前文件流为读模式或者写缓冲为空，则分配新的流缓冲区并调整read buff的位置  

**该函数经常会在无leak的heap利用题目中使用，我们通过修改IO\_write\_ptr的值，使其减去\IO\_write\_base的值为一个很大的值，从而打印出很多libc的信息。**

<pre class = "prettyprint lang-javascript">
_IO_new_file_overflow (_IO_FILE *f, int ch)
{
	if (f->_flags & _IO_NO_WRITES) /* SET ERROR */
	{
		f->_flags |= _IO_ERR_SEEN;
		__set_errno (EBADF);
		return EOF;
	}
	/* If currently reading or no buffer allocated. */
	if ((f->_flags & _IO_CURRENTLY_PUTTING) == 0 || f->_IO_write_base == NULL)
	{
		//通过置f->_flags为0x1000，从而绕过该检查。
		/* Allocate a buffer if needed. */
		if (f->_IO_write_base == NULL)
		{
			_IO_doallocbuf (f);
			_IO_setg (f, f->_IO_buf_base, f->_IO_buf_base, f->_IO_buf_base);
		}
		/* Otherwise must be currently reading.
		If _IO_read_ptr (and hence also _IO_read_end) is at the buffer end,
		logically slide the buffer forwards one block (by setting the
		read pointers to all point at the beginning of the block).  This
		makes room for subsequent output.
		Otherwise, set the read pointers to _IO_read_end (leaving that
		alone, so it can continue to correspond to the external position). */
		if (__glibc_unlikely (_IO_in_backup (f)))
		{
			//如果f设置了备份缓存，则替换该备份缓存区为主缓存区，并滑动读入缓冲区为写缓冲空出位置
			size_t nbackup = f->_IO_read_end - f->_IO_read_ptr;
			_IO_free_backup_area (f);
			f->_IO_read_base -= MIN (nbackup,f->_IO_read_base - f->_IO_buf_base);
			f->_IO_read_ptr = f->_IO_read_base;
		}
	
		if (f->_IO_read_ptr == f->_IO_buf_end)
			f->_IO_read_end = f->_IO_read_ptr = f->_IO_buf_base;
		f->_IO_write_ptr = f->_IO_read_ptr;
		f->_IO_write_base = f->_IO_write_ptr;
		f->_IO_write_end = f->_IO_buf_end;
		f->_IO_read_base = f->_IO_read_ptr = f->_IO_read_end;
		
		f->_flags |= _IO_CURRENTLY_PUTTING;
		if (f->_mode <= 0 && f->_flags & (_IO_LINE_BUF | _IO_UNBUFFERED))
			f->_IO_write_end = f->_IO_write_ptr;
	}
	if (ch == EOF)
		return <a href = "#11">_IO_do_write (f, f->_IO_write_base,f->_IO_write_ptr - f->_IO_write_base);</a>
		//能够进行信息泄露的关键点，经常在无leak的堆题中出现。
	if (f->_IO_write_ptr == f->_IO_buf_end ) /* Buffer is really full */
		if (_IO_do_flush (f) == EOF)
			return EOF;
	*f->_IO_write_ptr++ = ch;
	if ((f->_flags & _IO_UNBUFFERED) || ((f->_flags & _IO_LINE_BUF) && ch == '\n'))
		if (_IO_do_write (f, f->_IO_write_base,f->_IO_write_ptr - f->_IO_write_base) == EOF)
		return EOF;
	return (unsigned char) ch;
}
libc_hidden_ver (_IO_new_file_overflow, _IO_file_overflow)
</pre>
<a name = "11"></a>
## 3.3 new\_do\_write ##
该函数为\_IO\_do\_write函数的功能实现函数。
<pre class = "prettyprint lang-javascript">
static _IO_size_t new_do_write (_IO_FILE *fp, const char *data, _IO_size_t to_do)
{
	_IO_size_t count;
	if (fp->_flags & _IO_IS_APPENDING)
		/* On a system without a proper O_APPEND implementation, you would need to sys_seek(0, SEEK_END) here, but is not needed nor desirable for Unix- or Posix-like systems.Instead, just indicate that offset (before and after) is unpredictable. */
		fp->_offset = _IO_pos_BAD;
	else if (fp->_IO_read_end != fp->_IO_write_base)
	{
		//要想绕过该判断，则fp->_IO_read_end == fp->_IO_write_base
		_IO_off64_t new_pos = _IO_SYSSEEK (fp, fp->_IO_write_base - fp->_IO_read_end, 1);
		if (new_pos == _IO_pos_BAD)
			return 0;
		fp->_offset = new_pos;
	}
	count = _IO_SYSWRITE (fp, data, to_do);
	if (fp->_cur_column && count)
		fp->_cur_column = _IO_adjust_column (fp->_cur_column - 1, data, count) + 1;
	_IO_setg (fp, fp->_IO_buf_base, fp->_IO_buf_base, fp->_IO_buf_base);
	fp->_IO_write_base = fp->_IO_write_ptr = fp->_IO_buf_base;
	fp->_IO_write_end = (fp->_mode <= 0
		       && (fp->_flags & (_IO_LINE_BUF | _IO_UNBUFFERED))
		       ? fp->_IO_buf_base : fp->_IO_buf_end);
	return count;
}
</pre>
从源代码中可以看到，当完成将数据写回设备后，如果文件流为行缓冲或者无缓冲，则初始化文件流\_IO\_write\_end为\_IO\_buf\_base。否则，初始化文件流\_IO\_write\_end为\_IO\_buf\_end。
# 4. fclose #
**注：调用vtable列表中的\_\_finish函数指针**
<pre class="prettyprint lang-javascript"> 
int _IO_new_fclose (_IO_FILE *fp)
{
	int status;
	CHECK_FILE(fp, EOF);	//fp->flag & 0xffff0000 = 0xfdab0000
	
	#if SHLIB_COMPAT (libc, GLIBC_2_0, GLIBC_2_1)
	/* We desperately try to help programs which are using streams in a 
	strange way and mix old and new functions.  Detect old streams here.  */
	//该段代码的意义在于检测是否当前程序使用了老的文件流，如果是（fp->_vtable_off !=0)则调用_IO_old_fclose函数
		if (_IO_vtable_offset (fp) != 0)
			return _IO_old_fclose (fp);
	#endif
	/* First unlink the stream.  */
	if (fp->_IO_file_flags & _IO_IS_FILEBUF)	//_IO_IS_FILEBUF = 0x2000，表示当前fp文件流被打开过
		_IO_un_link ((struct _IO_FILE_plus *) fp);
	//将fp从_IO_list_all链表上拆下
	_IO_acquire_lock (fp);
	if (fp->_IO_file_flags & _IO_IS_FILEBUF)	//如果想要调用_close，则需要满足条件（fp->_flags & 0x2000 !=0 )
		status = _IO_file_close_it (fp);	//调用_IO_file_close_it函数关闭文件流，在这里会调用vtable的_close函数
	else
		status = fp->_flags & _IO_ERR_SEEN ? -1 : 0;
	_IO_release_lock (fp);
	_IO_FINISH (fp);	//调用vtable函数列表的__finish函数，如果想要走到这里，需要满足条件（fp->_flags & 0x2000 == 0）
	if (fp->_mode > 0)
	{
		/* This stream has a wide orientation.  This means we have to free
		the conversion functions.  */
		//宽字节流的转换功能的释放
		struct _IO_codecvt *cc = fp->_codecvt;

		__libc_lock_lock (__gconv_lock);
		__gconv_release_step (cc->__cd_in.__cd.__steps);
		__gconv_release_step (cc->__cd_out.__cd.__steps);
		__libc_lock_unlock (__gconv_lock);
	}
	else
	{
		//是否有备份
		if (_IO_have_backup (fp))	//_IO_save_base字段
			_IO_free_backup_area (fp);
	}
	if (fp != _IO_stdin && fp != _IO_stdout && fp != _IO_stderr)
	{
		fp->_IO_file_flags = 0;
		free(fp);
	}
	return status;
}
</pre>
该函数功能为：  
1. 首先判断是否使用了旧的文件流指针，如果是则直接调用\_IO\_old_fclose函数。  
2. 判断当前文件流指针是否被打开过（\_IO\_IS\_FILEBUF标志），**如果是则进行拆链和关闭文件流（实际调用了\_IO\_file\_close\_it函数），在该函数中又调用了vtable中的\_\_close函数指针。**  
3. **调用vtable中\_\_finish函数指针。**  
4. 宽字节流相关处理（通过\_mode字段判断）。  
5. 流备份（\_IO\_save\_base字段）相关处理。  
6. 如果fp指针不是标准流（stdin，stdoub，stderr），则释放该文件流。

<pre class = "prettyprint lang-javascript">
int _IO_new_file_close_it (_IO_FILE *fp)
{
  int write_status;
  if (!_IO_file_is_open (fp))	//判断fp->_fileno >= 0
    return EOF;

  if ((fp->_flags & _IO_NO_WRITES) == 0
      && (fp->_flags & _IO_CURRENTLY_PUTTING) != 0)
    write_status = _IO_do_flush (fp);
  else
    write_status = 0;

  _IO_unsave_markers (fp);
  int close_status = ((fp->_flags2 & _IO_FLAGS2_NOCLOSE) == 0
		      ? _IO_SYSCLOSE (fp) : 0);		//调用vtable的_close()
</pre>
在函数中能中黑色加粗的部分是我们可以利用的两个点，即：\_\_close和\_\_finish，需要注意的是，这两条利用路径的判断条件不同，如果我们想要利用\_\_close，则必须满足fp->\_flags & 0x2000 != 0；与此相反，如果我们想要利用\_\_finish，则必须保证fp->_flags & 0x2000 == 0。
