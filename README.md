# sisopmodul4

[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/V7fOtAk7)
|    NRP     |      Name      |
| :--------: | :------------: |
| 5025221000 | Student 1 Name |
| 5025221000 | Student 2 Name |
| 5025221000 | Student 3 Name |

# Praktikum Modul 4 _(Module 4 Lab Work)_

</div>

### Daftar Soal _(Task List)_

- [Task 1 - FUSecure](/task-1/)

- [Task 2 - LawakFS++](/task-2/)

- [Task 3 - Drama Troll](/task-3/)

- [Task 4 - LilHabOS](/task-4/)

### Laporan Resmi Praktikum Modul 4 _(Module 4 Lab Work Report)_



# **LAPORAN PRAKTIKUM FUSecure task 1**

---

## **1. Persiapan Direktori dan User**

### **a. Membuat User Baru**

Agar sistem bisa menguji akses user dengan benar, buat dua user:

```bash
sudo useradd yuadi
sudo passwd yuadi
sudo useradd irwandi
sudo passwd irwandi
```
---

### **b. Membuat Struktur Folder**

Buat folder utama dan subfolder:

```bash
sudo mkdir -p /home/shared_files/public
sudo mkdir -p /home/shared_files/private_yuadi
sudo mkdir -p /home/shared_files/private_irwandi
```

---

### **c. Mengatur Kepemilikan Folder**

Agar hanya user terkait yang bisa mengakses folder private di host (opsional, untuk keamanan):

```bash
sudo chown -R yuadi:yuadi /home/shared_files/private_yuadi
sudo chown -R irwandi:irwandi /home/shared_files/private_irwandi
```

---

### **d. Mengisi Folder dengan File Contoh**

* **Public:**

  ```bash
  echo "Materi Algoritma" | sudo tee /home/shared_files/public/materi_algoritma.txt
  ```
* **Private Yuadi:**

  ```bash
  sudo -u yuadi bash -c 'echo "Jawaban Praktikum 1" > /home/shared_files/private_yuadi/jawaban_praktikum1.txt'
  ```
* **Private Irwandi:**

  ```bash
  sudo -u irwandi bash -c 'echo "Jawaban Praktikum Sisop" > /home/shared_files/private_irwandi/tugas_sisop.txt'
  ```
---

## **2. Pembuatan Program FUSE**

### **a. Membuat File Source Code**

Buat file source code dengan editor teks di terminal:

```bash
nano fusys.c
```

Lalu copy-paste kode berikut:

```c
#define FUSE_USE_VERSION 28
#include <fuse.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/types.h>
#include <dirent.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <pwd.h>

static const char *src_dir = "/home/shared_files";

// Fungsi untuk mendapatkan UID user dari nama
uid_t get_uid_by_name(const char *username) {
    struct passwd *pwd = getpwnam(username);
    if (pwd) return pwd->pw_uid;
    return (uid_t)-1;
}

// Fungsi untuk mengecek akses user ke private folder
int check_access(const char *path, uid_t uid) {
    if (strncmp(path, "/private_yuadi", 14) == 0) {
        if (uid != get_uid_by_name("yuadi")) return 0; // hanya yuadi
    }
    if (strncmp(path, "/private_irwandi", 16) == 0) {
        if (uid != get_uid_by_name("irwandi")) return 0; // hanya irwandi
    }
    return 1; // boleh akses
}

static int fs_getattr(const char *path, struct stat *stbuf) {
    int res;
    char real_path[1024];
    sprintf(real_path, "%s%s", src_dir, path);

    if (!check_access(path, fuse_get_context()->uid)) return -EACCES;

    res = lstat(real_path, stbuf);
    if (res == -1) return -errno;
    return 0;
}

static int fs_readdir(const char *path, void *buf, fuse_fill_dir_t filler, off_t offset, struct fuse_file_info *fi) {
    char real_path[1024];
    sprintf(real_path, "%s%s", src_dir, path);
    DIR *dp = opendir(real_path);
    struct dirent *de;

    if (!check_access(path, fuse_get_context()->uid)) return -EACCES;
    if (dp == NULL) return -errno;

    while ((de = readdir(dp)) != NULL) {
        struct stat st;
        memset(&st, 0, sizeof(st));
        st.st_ino = de->d_ino;
        st.st_mode = de->d_type << 12;
        if (filler(buf, de->d_name, &st, 0)) break;
    }
    closedir(dp);
    return 0;
}

static int fs_open(const char *path, struct fuse_file_info *fi) {
    char real_path[1024];
    sprintf(real_path, "%s%s", src_dir, path);

    if (!check_access(path, fuse_get_context()->uid)) return -EACCES;

    if ((fi->flags & O_ACCMODE) != O_RDONLY)
        return -EACCES;
    int res = open(real_path, O_RDONLY);
    if (res == -1) return -errno;
    close(res);
    return 0;
}

static int fs_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    char real_path[1024];
    sprintf(real_path, "%s%s", src_dir, path);

    if (!check_access(path, fuse_get_context()->uid)) return -EACCES;

    int fd = open(real_path, O_RDONLY);
    if (fd == -1) return -errno;
    int res = pread(fd, buf, size, offset);
    if (res == -1) res = -errno;
    close(fd);
    return res;
}

// Semua operasi tulis di-block
static int fs_mkdir(const char *path, mode_t mode) { return -EACCES; }
static int fs_mknod(const char *path, mode_t mode, dev_t rdev) { return -EACCES; }
static int fs_unlink(const char *path) { return -EACCES; }
static int fs_rmdir(const char *path) { return -EACCES; }
static int fs_write(const char *path, const char *buf, size_t size, off_t offset, struct fuse_file_info *fi) { return -EACCES; }
static int fs_rename(const char *from, const char *to) { return -EACCES; }
static int fs_chmod(const char *path, mode_t mode) { return -EACCES; }
static int fs_chown(const char *path, uid_t uid, gid_t gid) { return -EACCES; }
static int fs_truncate(const char *path, off_t size) { return -EACCES; }
static int fs_create(const char *path, mode_t mode, struct fuse_file_info *fi) { return -EACCES; }

static struct fuse_operations fs_oper = {
    .getattr = fs_getattr,
    .readdir = fs_readdir,
    .open    = fs_open,
    .read    = fs_read,
    .mkdir   = fs_mkdir,
    .mknod   = fs_mknod,
    .unlink  = fs_unlink,
    .rmdir   = fs_rmdir,
    .write   = fs_write,
    .rename  = fs_rename,
    .chmod   = fs_chmod,
    .chown   = fs_chown,
    .truncate= fs_truncate,
    .create  = fs_create,
};

int main(int argc, char *argv[]) {
    umask(0);
    return fuse_main(argc, argv, &fs_oper, NULL);
}
```

![image](https://github.com/user-attachments/assets/9bcbd672-5335-4406-8cbe-37a7a0ee06de)

---

### **b. Meng-compile Program**

Pastikan sudah install `libfuse-dev`:

```bash
sudo apt update
sudo apt install libfuse-dev
```

Compile kode:

```bash
gcc -Wall `pkg-config fuse --cflags` fusys.c -o fusys `pkg-config fuse --libs`
```

![image](https://github.com/user-attachments/assets/e801cd6c-c1a1-46e9-b313-c552c3a2f673)

---

## **3. Menjalankan dan Menguji FUSecure**

### **a. Membuat Mount Point**

```bash
sudo mkdir -p /mnt/secure_fs
```

### **b. Menjalankan FUSE**

```bash
sudo ./fusys /mnt/secure_fs -o allow_other
```

![image](https://github.com/user-attachments/assets/23747329-9510-47be-99cd-7c95c0d6b625)

---

### **c. Pengujian dari Terminal Lain**

* **Cek isi root FUSE:**

  ```bash
  ls /mnt/secure_fs
  ```

  **Output:**

  ```
  public  private_yuadi  private_irwandi
  ```

![image](https://github.com/user-attachments/assets/a113724b-64c5-4576-98b4-10b636d01f28)

* **Tes akses file public:**

  ```bash
  cat /mnt/secure_fs/public/materi_algoritma.txt
  ```

  **Output:**

  ```
  Materi Algoritma
  ```

![image](https://github.com/user-attachments/assets/e0159270-3b50-4109-9a16-b155a2cb3c60)

* **Tes akses file private:**

  * Sebagai `Irwandi`:

    ```bash
    sudo -u irwandi cat /mnt/secure_fs/private_irwandi/tugas_sisop.txt
    ```

![image](https://github.com/user-attachments/assets/641fafac-45ec-4928-b8e8-2371dd5e7a82)

  * Sebagai `Yuadi`:

    ```bash
        sudo -u yuadi cat /mnt/secure_fs/private_yuadi/jawaban_praktikum1.txt
    ```

![image](https://github.com/user-attachments/assets/22426c99-4552-4e03-ba59-7e2994915d4c)

* **Tes akses file private user lain (ditolak):**

  ```bash
  sudo -u irwandi cat /mnt/secure_fs/private_yuadi/jawaban_praktikum1.txt
  ```

  **Output:**

  ```
  cat: /mnt/secure_fs/private_yuadi/jawaban_praktikum1.txt: Permission denied
  ```

  *\[Screenshot hasil permission denied]*

* **Tes operasi write (harus gagal):**

  ```bash
  touch /mnt/secure_fs/public/file_baru.txt
  ```

  **Output:**

  ```
  touch: cannot touch '/mnt/secure_fs/public/file_baru.txt': Permission denied
  ```

![image](https://github.com/user-attachments/assets/fd66ccec-196f-457f-926d-66f68d39e870)

---

### **d. Unmount FUSE Setelah Selesai**

```bash
sudo fusermount -u /mnt/secure_fs
```
---

## **4. Penjelasan Singkat Kode**

* File system hanya mengizinkan operasi baca untuk semua user.
* Folder `public` dapat diakses siapa saja, sedangkan folder `private_yuadi` dan `private_irwandi` hanya dapat diakses oleh user sesuai nama folder.
* Semua operasi write, create, delete, rename, chmod, chown, dsb. otomatis akan gagal (`return -EACCES`).

---

## **5. Kesimpulan**

Dengan langkah di atas, sistem file FUSecure berhasil dibuat dan diuji sesuai soal.
Semua aturan akses dan read-only berjalan dengan baik.

---
