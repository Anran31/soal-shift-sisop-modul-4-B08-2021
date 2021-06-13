# soal-shift-sisop-modul-4-B08-2021

## Soal 1

Di suatu jurusan, terdapat admin lab baru yang super duper gabut, ia bernama Sin. Sin baru menjadi admin di lab tersebut selama 1 bulan. Selama sebulan tersebut ia bertemu orang-orang hebat di lab tersebut, salah satunya yaitu Sei. Sei dan Sin akhirnya berteman baik. Karena belakangan ini sedang ramai tentang kasus keamanan data, mereka berniat membuat filesystem dengan metode encode yang mutakhir. Berikut adalah filesystem rancangan Sin dan Sei :

**a.** Jika sebuah direktori dibuat dengan awalan “AtoZ_”, maka direktori tersebut akan menjadi direktori ter-encode.

**b.** Jika sebuah direktori di-rename dengan awalan “AtoZ_”, maka direktori tersebut akan menjadi direktori ter-encode.

**c.** Apabila direktori yang terenkripsi di-rename menjadi tidak ter-encode, maka isi direktori tersebut akan terdecode.

**d.** Setiap pembuatan direktori ter-encode (mkdir atau rename) akan tercatat ke sebuah log. Format : /home/[USER]/Downloads/[Nama Direktori] → /home/[USER]/Downloads/AtoZ_[Nama Direktori]

**e.** Metode encode pada suatu direktori juga berlaku terhadap direktori yang ada di dalamnya.(rekursif)

#### Jawab
Untuk menyelesaikan soal no 1, maka hal yang harus dilakukan adalah melakukan perubahan pada fungsi readdir. Pada fungsi readdir, jika di dalam path folder yang akan dibuka terdapat folder yang berawalan `AtoZ_`, maka ketika membaca isi dari folder tersebut, seluruh nama file dan folder yang akan disimpan ke dalam folder akan terenkripsi berdasarkan Atbash Cypher.

```c
    static int xmp_readdir(const char *path, void *buf, fuse_fill_dir_t filler, off_t offset, struct fuse_file_info *fi)
    {
        char fpath[1024] = {0};
        int change = 0;
        change = getfpath(path,fpath,0);
        int res = 0;
        printf("readdir path = %s\nfpath = %s\n",path,fpath);
        DIR *dp;
        struct dirent *de;
        (void) offset;
        (void) fi;

        dp = opendir(fpath);

        if (dp == NULL) return -errno;

        while ((de = readdir(dp)) != NULL) {
            struct stat st;

            memset(&st, 0, sizeof(st));

            st.st_ino = de->d_ino;
            st.st_mode = de->d_type << 12;
            if(change)
            {
                char nameCopy[1024] = {0};
                strcpy(nameCopy,de->d_name);
                if(S_ISDIR(st.st_mode))
                {
                    Atbash(nameCopy,0);
                }
                else Atbash(nameCopy,1);
                res = (filler(buf, nameCopy, &st, 0));
                //printf("readdir nameCopy = %s\n",nameCopy);
            }
            else{
                res = (filler(buf, de->d_name, &st, 0));
                //printf("readdir de->d_name = %s\n",de->d_name);
            }
            if(res!=0) break;
        }

        closedir(dp);
        const char *desc[] = {path};
        fsLog("INFO","READDIR",1, desc);
        return 0;
    }
```

Untuk mengetahui apakah di path yang akan terbuka terdapat suatu folder yang berawalan `AtoZ_`, kami membuat suatu fungsi bernama `getfpath()` yang dimana akan mereturn sebuah nilai berupa 0 jika di dalam path tidak terdapat `AtoZ_` dan 1 jika sebaliknya. Nilai tersebut akan digunakan dalam perulangan while saat membaca suatu direktori dan jika nilainya satu, maka nama filenya akan dienkripsi menggunakan Atbash Cypher. Fungsi `getfpath()` juga digunakan untuk mendapatkan path dari folder asli yang digunakan.

```c
    int getfpath(const char *path, char *fpath, int isFile)
    {
        int change = 0;
        if(strcmp(path,"/") == 0)
        {
            path=dirpath;

            sprintf(fpath,"%s",path);
        }
        else
        {
            strcat(fpath,dirpath);
            char pathCopy[1024] = {0};
            strcpy(pathCopy,path);

            char *lastPos = strrchr(path,'/');
            *lastPos++;
            const char *p="/";
            char *a,*b;
            for( a=strtok_r(pathCopy,p,&b) ; a!=NULL ; a=strtok_r(NULL,p,&b) ) {
                if(!change)
                {
                    strcat(fpath,"/");
                    strcat(fpath,a);
                }
                else
                {
                    char changeName[1024] = {0};
                    strcpy(changeName,a);
                    if(!strcmp(changeName,lastPos) && isFile) Atbash(changeName,1);
                    else Atbash(changeName,0);
                    strcat(fpath,"/");
                    strcat(fpath,changeName);
                }
                if(strncmp(a,"AtoZ_",5) == 0) change = 1;
                //printf("a = %s change = %d\n",a,change);
            }
        }

        return change;
        printf("getfpath fpath = %s\n",fpath);
    }
```

Fungsi `getfpath()` melakukan iterasi pada seluruh folder yang ada di path menggunakan fungsi `strtok_r()` yang membagi path menjadi token berdasarkan tanda `/`, kemudian setiap token dicek apakan 5 huruf terawalnya merupakan `AtoZ_` atau bukan. Jika iya, maka token-token selanjutnya akan diubah menggunakan Atbash Cypher.

```c
    void Atbash(char *name, int isFile)
    {
        int change = 1;
        for(int i = 0; i<strlen(name); i++)
        {
            if(name[i] == '.' && isFile) change=0;
            if(change)
            {
                if(name[i]>='A'&&name[i]<='Z') name[i] = 'Z'+'A'-name[i];
                if(name[i]>='a'&&name[i]<='z') name[i] =  'z'+'a'-name[i];
            }
        }
    }
```

Fungsi `getfpath()` tidak hanya dipanggil pada `readdir` saja, tetapi hampir seluruh system call fuse yang lainnya juga.

Kemudian untuk poin selanjutnya, yaitu membuat log khusus ketika terjadi pembuatan atau rename direktori menjadi terenkripsi (awalan `AtoZ_`) akan dibuat sebuah log khusus. Hal tersebut dapat diselesaikan dengan membuat fungsi baru yaitu `AtoZLog()` yang akan membuat log tersebut. Fungsi `AtoZLog()` akan dipanggil pada system call fuse `mkdir` jika direktori yang dibuat terdapat awalan `AtoZ_` atau `rename` jika direktori dirubah namanya menjadi terdapat awalan `AtoZ_`.

```c
    void AtoZLog(char *cmd, char *firstPath, char *secondPath)
    {
        FILE *f = fopen(AtoZLogPath, "a");

        if(!strcmp(cmd,"RENAME"))
        {
            fprintf(f, "%s::%s -> %s\n", cmd, firstPath, secondPath);
        }
        else if(!strcmp(cmd,"MKDIR"))
        {
            fprintf(f, "%s::%s\n", cmd, firstPath);
        }

        fclose(f);
    }
```

```c
    static int xmp_rename(const char *from, const char *to)
    {
        char fullFrom[1024],fullTo[1024];
        if(strcmp(from,"/") == 0)
        {
            from=dirpath;
            sprintf(fullFrom,"%s",from);
        }
        else sprintf(fullFrom, "%s%s",dirpath,from);

        if(strcmp(to,"/") == 0)
        {
            to=dirpath;
            sprintf(fullTo,"%s",to);
        }
        else sprintf(fullTo, "%s%s",dirpath,to);

        int res;
        printf("rename from = %s to = %s\n",fullFrom, fullTo);
        res = rename(fullFrom, fullTo);
        DIR *dp = opendir(fullTo);
        char *lastSlash = strrchr(fullTo,'/');
        char *toAtoZ_ = strstr(lastSlash-1,"/AtoZ_");


        if(toAtoZ_ != NULL && dp != NULL) 
        {
            printf("a dir renamed to /AtoZ_\n");
            closedir(dp);
            AtoZLog("RENAME",fullFrom,fullTo);
        }
        if (res == -1)
            return -errno;
        
        const char *desc[] = {from,to};
        fsLog("INFO","RENAME",2, desc);
        return 0;
    }
```

```c
    static int xmp_mkdir(const char *path, mode_t mode)
    {
        char fpath[1024] = {0};
        getfpath(path,fpath,0);

        int res;
        char *lastSlash = strrchr(fpath,'/');
        char *toAtoZ_ = strstr(lastSlash-1,"/AtoZ_");

        res = mkdir(fpath, mode);
        if(toAtoZ_ != NULL) AtoZLog("MKDIR",fpath, "");
        if (res == -1)
        return -errno;

        const char *desc[] = {path};
        fsLog("INFO","MKDIR",1, desc);
        return 0;
    }
```

## Soal 2

Selain itu Sei mengusulkan untuk membuat metode enkripsi tambahan agar data pada komputer mereka semakin aman. Berikut rancangan metode enkripsi tambahan yang dirancang oleh Sei

**a.** Jika sebuah direktori dibuat dengan awalan “RX_[Nama]”, maka direktori tersebut akan menjadi direktori terencode beserta isinya dengan perubahan nama isi sesuai kasus nomor 1 dengan algoritma tambahan ROT13 (Atbash + ROT13).

**b.** Jika sebuah direktori di-rename dengan awalan “RX_[Nama]”, maka direktori tersebut akan menjadi direktori terencode beserta isinya dengan perubahan nama isi sesuai dengan kasus nomor 1 dengan algoritma tambahan Vigenere Cipher dengan key “SISOP” (Case-sensitive, Atbash + Vigenere).

**c.** Apabila direktori yang terencode di-rename (Dihilangkan “RX_” nya), maka folder menjadi tidak terencode dan isi direktori tersebut akan terdecode berdasar nama aslinya.

**d.** Setiap pembuatan direktori terencode (mkdir atau rename) akan tercatat ke sebuah log file beserta methodnya (apakah itu mkdir atau rename).

**e.** Pada metode enkripsi ini, file-file pada direktori asli akan menjadi terpecah menjadi file-file kecil sebesar 1024 bytes, sementara jika diakses melalui filesystem rancangan Sin dan Sei akan menjadi normal. Sebagai contoh, Suatu_File.txt berukuran 3 kiloBytes pada directory asli akan menjadi 3 file kecil yakni:

Suatu_File.txt.0000
Suatu_File.txt.0001
Suatu_File.txt.0002

Ketika diakses melalui filesystem hanya akan muncul Suatu_File.txt

## Soal 3


Karena Sin masih super duper gabut akhirnya dia menambahkan sebuah fitur lagi pada filesystem mereka. 

**a.** Jika sebuah direktori dibuat dengan awalan “A_is_a_”, maka direktori tersebut akan menjadi sebuah direktori spesial.

**b.** Jika sebuah direktori di-rename dengan memberi awalan “A_is_a_”, maka direktori tersebut akan menjadi sebuah direktori spesial.

**c.** Apabila direktori yang terenkripsi di-rename dengan menghapus “A_is_a_” pada bagian awal nama folder maka direktori tersebut menjadi direktori normal.

**d.** Direktori spesial adalah direktori yang mengembalikan enkripsi/encoding pada direktori “AtoZ_” maupun “RX_” namun masing-masing aturan mereka tetap berjalan pada direktori di dalamnya (sifat recursive  “AtoZ_” dan “RX_” tetap berjalan pada subdirektori).

**e.** Pada direktori spesial semua nama file (tidak termasuk ekstensi) pada fuse akan berubah menjadi lowercase insensitive dan diberi ekstensi baru berupa nilai desimal dari binner perbedaan namanya.

Contohnya jika pada direktori asli nama filenya adalah “FiLe_CoNtoH.txt” maka pada fuse akan menjadi “file_contoh.txt.1321”. 1321 berasal dari biner 10100101001.

## Soal 4

Untuk memudahkan dalam memonitor kegiatan pada filesystem mereka Sin dan Sei membuat sebuah log system dengan spesifikasi sebagai berikut.

**a.** Log system yang akan terbentuk bernama “SinSeiFS.log” pada direktori home pengguna (/home/[user]/SinSeiFS.log). Log system ini akan menyimpan daftar perintah system call yang telah dijalankan pada filesystem.

**b.** Karena Sin dan Sei suka kerapian maka log yang dibuat akan dibagi menjadi dua level, yaitu INFO dan WARNING.

**c.** Untuk log level WARNING, digunakan untuk mencatat syscall rmdir dan unlink.

**d.** Sisanya, akan dicatat pada level INFO.

**e.** Format untuk logging yaitu:


[Level]::[dd][mm][yyyy]-[HH]:[MM]:[SS]:[CMD]::[DESC :: DESC]

Level : Level logging, dd : 2 digit tanggal, mm : 2 digit bulan, yyyy : 4 digit tahun, HH : 2 digit jam (format 24 Jam),MM : 2 digit menit, SS : 2 digit detik, CMD : System Call yang terpanggil, DESC : informasi dan parameter tambahan

INFO::28052021-10:00:00:CREATE::/test.txt
INFO::28052021-10:01:00:RENAME::/test.txt::/rename.txt

### Jawab

Untuk menyelesaikan soal ini, maka hanya perlu membuat satu fungsi `fsLog()` yang akan dipanggil pada setiap system call yang dipanggil. fungsi `fsLog()` menerima parameter Level, command, jumlah deskripsi tambahan, serta deskripsi yang diperlukan.

```c
    void fsLog(char *level, char *cmd,int descLen, const char *desc[])
    {
        FILE *f = fopen(FSLogPath, "a");
        time_t now;
        time ( &now );
        struct tm * timeinfo = localtime (&now);
        fprintf(f, "%s::%s::%02d%02d%04d-%02d:%02d:%02d",level,cmd,timeinfo->tm_mday,timeinfo->tm_mon+1,timeinfo->tm_year+1900,timeinfo->tm_hour, timeinfo->tm_min, timeinfo->tm_sec);
        for(int i = 0; i<descLen; i++)
        {
            fprintf(f,"::%s",desc[i]);
        }
        fprintf(f,"\n");
        fclose(f);
    }
```

contoh pemanggilan pada rmdir:

```c
    static int xmp_rmdir(const char *path)
    {
        char fpath[1024] = {0};
        getfpath(path,fpath,0);
        int res;

        res = rmdir(fpath);
        if (res == -1)
        return -errno;

        const char *desc[] = {path};
        fsLog("WARNING","RMDIR",1, desc);
        return 0;
    }
```