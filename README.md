# SequoiaCam
---
title: Lectura y decodificación de los metadatos en imágenes de la cámara Sequoia
author: "Joan Cano"
output:
  html_document: default
  pdf_document: default
---
## Tipos de metadatos en imágenes

Los metadatos son datos que nos proporcionan información sobre los datos, entre ellos los descriptivos, estructurales y administrativos.
Estos pueden ser escritos en iḿagenes como las que toma nuestra cámara, que ayudarán en el proceso posterior de producción a partir de palabras clave.

Las imágenes pueden contener diferentes metadatos, regidos por las normas creadas por organizaciones encargadas de ello; algunas de estas normas son:

- IPTC
- IPTC para XMP
- **XMP**
- **EXIF**
- Dublin Core
- PLUS
- VRA

## Librerías de lectura de metadatos: Exiftools
Al haber tratado ya los metadatos de las imágenes de la [Sequoia](http://tecnitop.com/es/camaras-accesorios-y-software/), avanzo la necesidad de interpretar metadatos del estandar [XMP](https://en.wikipedia.org/wiki/Extensible_Metadata_Platform#Free_software_and_open-source_tools_.28read.2Fwrite_support.29), una marca registrada por Adobe Systems que cuando pasó a ser ISO, dejó se ser propietaria.

Por lo tanto, necesitaremos buscar un software (libre) capaz de leer XMP, a continuación los que he probado:    
- [GIMP](https://www.gimp.org/)    
- [digiKam](https://www.digikam.org/)    
- [ExifTool](https://www.sno.phy.queensu.ca/~phil/exiftool/)    
- [ImageMagick](https://www.imagemagick.org/script/index.php)

GIMP y ImageMagick vienen por defecto en Debian. Sin embargo Exiftool lo tenemos que instalar:  


```{bash, eval=FALSE}
# instalar exiftool
 apt-cache install libimage-exiftool-perl
```

# Mostrar los metadatos de una imagen

```{bash, eval=FALSE}
exiftool imagen.tif
```

ImageMagick también se puede ejecutar desde la terminal:
```{bash, eval=FALSE}
identify -verbose imagen.tif
```

[Resultados exiftool](/multispectralcam/exiftool.txt), [Resultados ImageMagick](/multispectralcam/imageMagick.txt)

***
### Lectura de los metadatos de una imagen:
```{bash, eval=FALSE}
system ("exiftool /home/joan/multispectralcam/test/IMG_170825_082736_0004_NIR.TIF")
```

### Exportar metadatos a csv
```{r}
output <- system ( "exiftool -n -csv /home/joan/multispectralcam/test/IMG_170825_082736_0004_NIR.TIF" , intern = TRUE ) 
df <- read.csv ( textConnection ( output ), stringsAsFactors = FALSE )
write.csv(df, file = "/home/joan/Escritorio/test.csv")
```

### Decodificar formato Base64 mediante Python
Se pueden apreciar metadatados muy valiosos para la investigación, como:
1. Yaw, pitch y roll
2. Viñeteado
3. Valor de luminosidad
4. Calibraciones
5. Corriente oscura
6. etc...    

Pero.. ¿Y la etiqueta **IrradianceList**?
Esta etiqueta viene codificada en **Base64**, un sistema de numeración posicional que utiliza el 64 como base y carácteres ASCII (usa el rango entre A-Z, a-z y 0-9, más dos símbolos que pueden variar). De manera que tenemos la siguiente cadena de carácteres:

> U5UwEQAAAABaBa4CAQBkAHAJ6sLRfAHBo95bwBegMxEAAAAAnAXQAgEAZACLX+zCj/G+wImc0b/
rrDYRAAAAAKIF0wIBAGQAltDqwi1DusAigqS/M8I5EQAAAADNBGgCAQBkANc958JGokfBxkHAPsbZ
PBEAAAAA+AP9AQEAZACYTeXCGYhEwV7Bsb833z8RAAAAAOQD8gEBAGQA21LqwrSgfcHvtz3ACeVCE
QAAAADzA/oBAQBkAICY+cK6j6HBPJaJQLL3RREAAAAAbAO3AQEAZAC0rQPD67Hbwa0xNEE+CUkRAA
AAAM8D6AEBAGQAS8j8wj+alsH7+SrBeA1MEQAAAAAPBYkCAQBkAA/iBMP/TBjCafi/wPkFTxEAAAAA
rgRZAgEAZABL0ATDLYzwwSFcr78KF1IRAAAAAI0FyAIBAGQAfjkBw6Iwc8FEj6LAcyRVEQAAAABsBb
gCAQBkAJ/d+8K8cFbBgX6owA==

***
Para poder decodificarlo se han utilizado una serie de librerías en [Python](https://www.python.org/):

- [exiftool](https://smarnach.github.io/pyexiftool/)
- [base64](https://docs.python.org/2/library/base64.html)
- [struct](https://docs.python.org/2/library/struct.html)

Este es el script:

```{python, eval=FALSE}
import exiftool
import base64
import struct
import glob
import os

tag = "XMP:IrradianceList"

directorio = 'test'
canales = ['GRE','RED','REG','NIR']

index = 0

for canal in canales:
    files = glob.glob(os.path.join(directorio, '*' + canal + '*'))

    with exiftool.ExifTool() as et:
        metadata = et.get_tag_batch(tag, files)

        #print metadata
        for file_metadata in metadata:
            decoded = base64.b64decode(file_metadata)

        index += 1

        for i in range(0, len(decoded), 28):
            print struct.unpack("qHHHHfff", decoded[i:i+28])

```

***
## Resultados
```{r, results='asis'}
irradianceList <-read.table("/home/joan/multispectralcam/decode.csv", header= T, sep=",")
knitr::kable(irradianceList)
```

*** 

Como se puede apreciar, aparecen muchas filas de datos, sin embargo solo necesitamos 1, la última obtenida por imagen. Las filas pueden variar según la imagen, esto es debido a que parece que la cámara guarda información a 5Hz.
