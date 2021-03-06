# Свойства файлов

## Сведения о файле

### Структура `stat`

С каждым файлом в файловой системе связана метаинформация (status), которая определяется структурой `struct stat`:

```
struct stat {
   dev_t     st_dev;         /* ID of device containing file */
   ino_t     st_ino;         /* Inode number */
   mode_t    st_mode;        /* File type and mode */
   nlink_t   st_nlink;       /* Number of hard links */
   uid_t     st_uid;         /* User ID of owner */
   gid_t     st_gid;         /* Group ID of owner */
   dev_t     st_rdev;        /* Device ID (if special file) */
   off_t     st_size;        /* Total size, in bytes */
   blksize_t st_blksize;     /* Block size for filesystem I/O */
   blkcnt_t  st_blocks;      /* Number of 512B blocks allocated */

   struct timespec st_atim;  /* Time of last access */
   struct timespec st_mtim;  /* Time of last modification */
   struct timespec st_ctim;  /* Time of last status change */

   /* Backward compatibility */
   #define st_atime st_atim.tv_sec      
   #define st_mtime st_mtim.tv_sec
   #define st_ctime st_ctim.tv_sec
};

```

Метаинформацию о файле получается с помощью команды `stat ИМЯ_ФАЙЛА` или одного из системных вызовов:
 * `int stat(const char *file_name, struct stat *stat_buffer)` - получение информации о файле по его имени;
 * `int fstat(int fd, struct stat *stat_buffer)` - то же самое, но для открытого файлового дескриптора;
 * `int lstat(const char *path_name, struct stat *stat_buffer)` - аналогично `stat`, но в случае, если имя файла указывает на символическую ссылку, то возвращается информация о самой ссылке, а не о файле, на который она ссылается.


### Режим доступа и типы файлов в POSIX

В POSIX есть несколько основных типов файлов:

 * Регулярный файл (`S_IFREG = 0100000`). Занимает место на диске, содержит самые обычные данные.
 * Каталог (`S_IFDIR = 0040000`). Файл специального типа, который хранит список имен файлов.
 * Символическая ссылка (`S_IFLNK = 0120000`). Файл, который ссылается на другой файл (в том числе в другом каталоге или даже на другой файловой системе), и с точки зрения фукнций ввода-вывода ничем не отличается от того файла, на который он ссылается.
 * Блочные (`S_IFBLK = 0060000`) и символьные (`S_IFCHR = 0020000`) устройства. Используются как удобный способ взаимодействия с оборудованием.
 * Именованные каналы (`S_IFIFO = 0010000`) и сокеты (`S_IFSOCK = 0140000`), предназначенные для межпроцессного взаимодействия.


Тип файла закодирован в одном поле структуры с режимом доступа (`rwxrwxrwx`) - целочисленном `.st_mode`.

Для выделения отдельных типов файлов применяются поразрядные операции с помощью одного из макросов: `S_ISREG(m)`, `S_ISDIR(m)`, `S_ISCHR(m)`, `S_ISBLK(m)`, `S_ISFIFO(m)`, `S_ISLNK(m)` и `S_ISSOCK(m)`, которые возвращают `0` в качестве false, и произвольное ненулевое значение в качестве true.

Для выделения режима доступа, который закодирован в младших битах `.st_mode`, которые можно извлекать, применяя поразрядные операции с помощью констант `S_IWUSR`, `S_IRGRP`, `S_IXOTH` и др. Полный список констант можно посмотреть в `man 7 inode`.

### Доступ к файлу

У каждого файла, помимо режима доступа (`rwx` для владельца, группы и остальных) есть два идентификатора, - целых положительных числа:
 * `.st_uid` - идентификатор пользователя-владельца файла;
 * `.st_gid` - идентификатор группы-владельца файла.

Права доступа "для владельца" применяются при совпадении идентификатора текущего пользователя (можно получить с помощью `getuid()`) с полем `.st_uid`. Аналогично - для группы при совпадении `getgid()` с `.st_gid`. В остальных случаях действуют права доступа "для остальных".

Удобным способом определения прав у текущего пользователя является использование системного вызова `access`:
```
int access(const char *path_name, int mode)
```

Этот системный вызов принимает в качестве параметра `mode` поразрядную комбинацию из флагов `R_OK`, `W_OK`, `X_OK` и `F_OK`, - соответственно способность чтения, записи, выполнения файла, и его существование. Возвращает 0 в случае, если перечисленные аттрибуты допустимы для текущего пользователя, и -1 в противном случае.


## Маска создания файлов

При создании новых файлов с помощью системного вызова `open` (и всех высокоуровневых функций, которые используют `open`), нужно обязательно указывать режим доступа для вновь созданных файлов.

В реальности режим доступа может отличаться от запрошенного: для вновь созданного файла (или каталога) применяется *маска создания файлов* с помощью поразрядной операции "и-не":
```
 /* Пусть umask = 0222 */
 open("new_file", O_WRONLY|O_CREAT, 0666); // OK

 /* Создан файл с аттрибутами 0666 & ~0222 = 0444 */
```

По умолчанию маска создания файлов равна `0000`, то есть не накладывает никаких ограничений. Системный вызов `umask` позволяет задать явным образом новую маску, которая может быть использована для предотвращения случайного создания файлов со слишком слабозащищенными правами доступа.



## Ссылки и файловые дескрипторы

Пара значений, хранящихся в полях `.st_dev` и `.st_ino`, позволяют однозначно идентифицировать любой файл в виртуальной файловой системе до её перезагрузки или отмонтирования каких-то частей файловой системы.

Поле `.st_nlink` хранит количество имен, связанных с данным именем файла, причем они могут находиться в разных каталогах одной физической файловой системы. Для файла, который существует в файловой системе, `.st_nlink` всегда не меньше `1`, при удалении файла это число становится равным `0`. Но если файл открыт хотя бы одним процессом в системе, то физически он не удаляется (хотя по имени его уже не найти), и доступен по файловому дескриптору до момента закрытия файла.

Удалить программным спосособом файл можно с помощью системного вызова `unlink`. Как следует из названия, этот системный вызов уменьшает количество ссылок `.st_nlink`.

Для предотвращения *состояния гонки* (race condition), у многих системных вызовов для работы с файлами существуют аналоги, подразумевающие работу с файловыми дескрипторами, а не с именами файлов.

Пример состояния гонки:
```
struct stat st;
assert(0==stat("my_file.txt", &st)); // OK

/* теперь кто-нибудь извне успевает удалить или
   переименовать файл между моментами этих двух вызовов */

int fd = open("my_file.txt", O_RDONLY); // ERR: файл не найден
/* ??? как же так? Только что был здесь, а теперь недоступен */
```

Если немного переделать этот пример с использованием `fstat`, то проблема решена:
```
int fd = open("my_file.txt", O_RDONLY);
if (-1!=fd) {

    /* Теперь кто-нибудь удаляет файл, или переименовывает
    его. Но нас это уже не должно волновать: файл существует
    до того момента, пока мы его не закроем */

    struct stat st;    
    fstat(fd, &st);
    off_t file_size = st.st_size; // размер доступных данных
}
```

## Разные полезные системные вызовы

* Добавление нового имени (увеличение ссылок) - `man 2 link`
* Удаление (уменьшение ссылок) - `man 2 unlink`
* Создание символической ссылки - `man 2 symlink`
* Чтение значения символической ссылки - `man 2 readlink`
* Создание каталога - `man 2 mkdir`
* Удаление пустого каталога - `man 2 rmdir`
* Изменение режима доступа - `man 2 chmod`
* Перемещение (переименование) файла - `man 2 rename`


## Аттрибуты файловых дескрипторов

Системный вызов `fcntl` предназначен для управления открытыми файлыми дескрипторами.

```
int fcntl(int fd, int get_command);
int fcntl(int fd, int set_command, void *set_value);
```

Для открытых файлов можно командами `F_GETFL` и `F_SETFL` получать и менять аттрибуты открытия: `O_APPEND`, `O_ASYNC`, `O_NONBLOCK`. В Linux не возможно изменить режим открытия файлов (например поменять `O_RDONLY` на `O_RDRW`), хотя в некоторых UNIX системах это допускается.

## Блокировки файлов

Проблема состояния гонки относится не только к возможным изменениям аттрибутов файлов, но и к проблеме *гонки данных* (data race) между различными процессами, которые хотят одновременно обращаться к одним и тем же файлам.

В UNIX-подобных системах существует два основных механизма для блокировки файлов.

### Блокировки BSD flock

Поддерживается \*BSD и Linux. Системный вызов `flock` имеет
сигнатуру:
```
int flock(int fd, int operation);
```
`fd` - открытый файловый дескриптор, возможные операции:
 * `LOCK_SH` - получить разделяемую блокировку. Разделяемую блокировку на файл могут накладывать разные процессы, не мешая друг другу, когда им требуется только читать данные из файла.
 * `LOCK_EX` - получить эксклюзивную блокировку. Только один процесс может это сделать.
 * `LOCK_UN` - разблокировать файла.

Блокировка накладывается не на файловые дескрипторы, а на сами файлы, с с которыми они связаны. При попытке получить блокировку (любого типа) к файлу, который кем-то эксклюзивно заблокирован, приведет к приостановке текущего процесса до тех пора, пока блокировка не будет снята.

### Блокировки POSIX file lock

Использует команды `F_GETLK` (получить блокировки), `F_SETLK` (установить блокиовку) и `F_SETLKW` (установить блокировку, но сначала дождаться, когда это возможно) системного вызова `fcntl` для управления блокировками.

В качестве третьего аргумента `fcntl` передается структура:
```
struct flock {
    off_t   l_start;  /* starting offset	*/
    off_t   l_len;    /* len = 0	means until end	of file	*/
    pid_t   l_pid;    /* lock owner */
    short   l_type;   /* lock type: read/write, etc. */
    short   l_whence; /* type of	l_start	*/
    int     l_sysid;  /* remote system id or zero for local */
};
```

В отличии от BSD `flock`, этот способ является более гибким, и позволяет управлять блокировками отдельных частей файла.

Пример:
```
struct flock fl;
memset(&fl, 0, sizeof(fl));

// Установить блокировку только для чтения (не эксклюзивно)
fl.l_type = F_RDLCK;

// Блокировка на весь файл
fl.l_whence = SEEK_SET; // по аналогии с lseek
fl.l_start = 0;         // по аналогии с lseek
fl.l_len = 0;           // 0 - до конца, либо размер области                        

// Установить блокировку для чтения. Если какой-то процесс
// уже захватил файл для записи, то ожидание, пока освободится
fcntl(fd, F_SETLKW, &fl);

// Блокировка на запись с 10 по 15 байты файла
fl.l_type = F_WRLCK;
fl.l_start = 10;
fl.l_len = 5;
fcntl(fd, F_SETLK, &fl);

// Снять блокировку с этого же диапазона
fl.l_type = F_UNLCK;
fcntl(fd, F_SETLK, &fl);

// При закрытии файла процессом все блокировки недействительны
close(fd);
```

### Блокировка BSD lockf

Упрощенным аналогом для блокировки отдельных частей файла является системный вызов `lockf`; для указания начала области блокировки нужно переместить указатель `lseek`.

```
int lockf(int fd, int cmd, off_t len);
```
В отличии от `fcntl`, этот системный вызов подразумевает только эксклюзивную блокировку (для записи).

### Команды flock и lslocks

Команда `flock` позоляет установить блокировку на произвольный файл для запуска какой-либо программы, и снимает блокировку при завершении работы дочернего процесса.

Команда `lslocks` отображает таблицу файлов с блокировками.
