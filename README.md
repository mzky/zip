

# 带压缩密码的多文件压缩解压完整例子
支持文件和目录压缩
```
package main

import (
	"bytes"
	"errors"
	"fmt"
	"io"
	"os"
	"path/filepath"
	"strings"

	"github.com/mzky/zip"
)


func main() {
	var array []string
	array = append(array, "/mnt/keepalived-1.4.5.tar.gz") //　待压缩文件
	if fileList, err := FileListFromPath(filepath.Dir("/mnt/popt-1.18/")); err == nil {//　待压缩目录
		array = append(array, fileList...)
	}
	fmt.Println("待压缩低文件列表：","\n"+strings.Join(array,"\n"))

	err := Zip("/mnt/demo.zip", "abc123~!@", array)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println("完成压缩！")

	// err = UnZip("/mnt/demo.zip", "1", "./") //解压相对路径
	err = UnZip("/mnt/demo.zip", "abc123~!@", "/")
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println("解压完成！")
}


func IsZip(zipPath string) bool {
	f, err := os.Open(zipPath)
	if err != nil {
		return false
	}
	defer f.Close()

	buf := make([]byte, 4)
	if n, err := f.Read(buf); err != nil || n < 4 {
		return false
	}

	return bytes.Equal(buf, []byte("PK\x03\x04"))
}

// FileIsExist 判断文件夹或文件是否存在, true为存在
func FileIsExist(fp string) bool {
	_, err := os.Stat(fp)
	return err == nil || os.IsExist(err)
}

// FileListFromPath 获取文件夹下文件列表，支持通配符*
func FileListFromPath(fp string) ([]string, error) {
	var fileList []string
	var err error
	if strings.Contains(fp, "*") {
		fileList, err = filepath.Glob(fp)
	} else {
		fileList, err = filepath.Glob(filepath.Join(fp, "*"))
	}

	for _, v := range fileList {
		if !IsFile(v) {
			fileList = TrimValueFromArray(fileList, v) // 删除此目录项
			fl, _ := FileListFromPath(v)
			fileList = append(fileList, fl...)
		}
	}
	return fileList, err
}

// IsFile 判断是文件还是目录
func IsFile(fp string) bool {
	fi, e := os.Stat(fp)
	if e != nil {
		return false
	}
	return !fi.IsDir()
}

// TrimValueFromArray 去除数组中指定元素
func TrimValueFromArray(strArray []string, trimValue string) []string {
	newArray := make([]string, 0)
	for _, v := range strArray {
		if strings.TrimSpace(trimValue) != strings.TrimSpace(v) {
			newArray = append(newArray, strings.TrimSpace(v))
		}
	}

	return newArray
}

// Zip password值可以为空""
func Zip(zipPath, password string, fileList []string) error {
	fz, err := os.Create(zipPath)
	if err != nil {
		return err
	}
	defer fz.Close()
	zw := zip.NewWriter(fz)
	defer zw.Close()

	for _, fileName := range fileList {
		fr, errA := os.Open(fileName)
		if errA != nil {
			return errA
		}

		// 写入文件的头信息
		var w io.Writer
		var errB error
		if password != "" {
			w, errB = zw.Encrypt(fileName, password, zip.AES256Encryption)
		} else {
			w, errB = zw.Create(fileName)
		}

		if errB != nil {
			return errB
		}

		// 写入文件内容
		_, errC := io.Copy(w, fr)
		if errC != nil {
			return errC
		}
		fr.Close()
	}
	return zw.Flush()
}

// UnZip password值可以为空""
// 当decompressPath值为"./"时，解压到相对路径
func UnZip(zipPath, password, decompressPath string) error {
	if !FileIsExist(zipPath){
		return errors.New("找不到压缩文件")
	}
	if !IsZip(zipPath) {
		return errors.New("压缩文件格式不正确或已损坏")
	}
	r, err := zip.OpenReader(zipPath)
	if err != nil {
		return err
	}
	defer r.Close()

	for _, f := range r.File {
		if password != "" {
			if f.IsEncrypted() {
				f.SetPassword(password)
			} else {
				return errors.New("must be encrypted")
			}
		}

		fp := filepath.Join(decompressPath, f.Name)
		_ = os.MkdirAll(filepath.Dir(fp), os.ModePerm)

		w, errA := os.Create(fp)
		if errA!=nil{
			return errors.New("无法创建解压文件")
		}
		fr, errB := f.Open()
		if errB!=nil{
			return errors.New("解压密码不正确")
		}
		if _, errC := io.Copy(w, fr); errC != nil {
			return errC
		}
		fr.Close()
		w.Close()
	}
	return nil
}
```

# 原版说明：

This fork add support for Standard Zip Encryption.

The work is based on https://github.com/alexmullins/zip

Available encryption:

```
zip.StandardEncryption
zip.AES128Encryption
zip.AES192Encryption
zip.AES256Encryption
```

## Warning

Zip Standard Encryption isn't actually secure.
Unless you have to work with it, please use AES encryption instead.

## Example Encrypt Zip

```
package main

import (
	"bytes"
	"io"
	"log"
	"os"

	"github.com/yeka/zip"
)

func main() {
	contents := []byte("Hello World")
	fzip, err := os.Create(`./test.zip`)
	if err != nil {
		log.Fatalln(err)
	}
	zipw := zip.NewWriter(fzip)
	defer zipw.Close()
	w, err := zipw.Encrypt(`test.txt`, `golang`, zip.AES256Encryption)
	if err != nil {
		log.Fatal(err)
	}
	_, err = io.Copy(w, bytes.NewReader(contents))
	if err != nil {
		log.Fatal(err)
	}
	zipw.Flush()
}
```

## Example Decrypt Zip

```
package main

import (
	"fmt"
	"io/ioutil"
	"log"

	"github.com/yeka/zip"
)

func main() {
	r, err := zip.OpenReader("encrypted.zip")
	if err != nil {
		log.Fatal(err)
	}
	defer r.Close()

	for _, f := range r.File {
		if f.IsEncrypted() {
			f.SetPassword("12345")
		}

		r, err := f.Open()
		if err != nil {
			log.Fatal(err)
		}

		buf, err := ioutil.ReadAll(r)
		if err != nil {
			log.Fatal(err)
		}
		defer r.Close()

		fmt.Printf("Size of %v: %v byte(s)\n", f.Name, len(buf))
	}
}
```

