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



Example
```
package main

import (
	"bytes"
	"fmt"
	"io"
	"os"
	"path/filepath"

	"github.com/mzky/zip"
)

func main() {
	var array []string
	array = append(array, "/mnt/k/v")
	array = append(array, "/mnt/a/b")
	err := Zip("/mnt/demo.zip", "1", array)
	if err != nil {
		fmt.Println(err)
	}

	err = UnZip("/mnt/demo.zip", "1", "/")
	if err != nil {
		fmt.Println(err)
	}

	fmt.Println(IsZip("/mnt/demo.zip"))
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

func Zip(zipPath, password string, fileList []string) error {
	fz, err := os.Create(zipPath)
	if err != nil {
		return err
	}
	zw := zip.NewWriter(fz)
	defer zw.Close()

	for _, fileName := range fileList {
		fr, err := os.Open(fileName)
		if err != nil {
			return err
		}

		// 写入文件的头信息
		var w io.Writer
		if password != "" {
			w, err = zw.Encrypt(fileName, password, zip.AES256Encryption)
		} else {
			w, err = zw.Create(fileName)
		}

		if err != nil {
			return err
		}

		// 写入文件内容
		_, err = io.Copy(w, fr)
		if err != nil {
			return err
		}
	}
	return zw.Flush()
}

func UnZip(zipPath, password, decompressPath string) error {
	r, err := zip.OpenReader(zipPath)
	if err != nil {
		return err
	}
	defer r.Close()

	for _, f := range r.File {
		if f.IsEncrypted() {
			f.SetPassword(password)
		}
		fp := filepath.Join(decompressPath, f.Name)
		dir, _ := filepath.Split(fp)
		err = os.MkdirAll(dir, os.ModePerm)
		if err != nil {
			return err
		}

		w, err := os.Create(fp)
		if nil != err {
			return err
		}

		fr, err := f.Open()
		if err != nil {
			return err
		}

		_, err = io.Copy(w, fr)
		if err != nil {
			return err
		}
		w.Close()
	}
	return nil
}
```
