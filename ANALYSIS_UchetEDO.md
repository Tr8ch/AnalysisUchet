# Анализ работы УчетЭДО

> \*Данный текст не является частью анализа и нужно удалить при сдаче.
> 
> Результатом анализа является ответ на вопрос:
> - Как интегрировать подписи системы, так чтобы они могли проверяться на check.doodocs.kz?
>

 - **Какой тип подписи? - XML, CMS, etc.**
	
		XML подпись

 - **Как выглядит подписанный документ, который подтверждает факт успешного подписания?**
		
		Это .zip архив, который называется всегда doument.zip, а файлы внутри называются как их UUID на сервисе.

  		Далее может быть 2 случая, если загружаете pdf и если загружаете другой формат.

		Если pdf:
			Архив содержит папку document, в которой находятся:
				Pdf файлы <названия>: 
				- Сведения о документе  <UUID_info.pdf>
				- Оригинал документа    <UUID.pdf>
				- Печатная форма        <UUID_print.pdf>
				- XML файл попдипсей    <UUID.xml>
- _Пример:_

	![PdfFiles](https://github.com/Tr8ch/AnalysisUchet/blob/main/images/filesUchetEdo.png)

		Если другой формат:
			Архив содержит 3 файла:
				- Оригинал документа    <UUID.(расширения файла: odt, docx...)>
				- Сведения о документе  <UUID_info.pdf>
				- XML файл подписей     <UUID.xml>
- _Пример:_

	![NonPdfFiles](https://github.com/Tr8ch/AnalysisUchet/blob/main/images/NonPdfFiles.png)
			
 - **Что конкретно подписывается? - Хеш, бинарные данные, UUID документ и т.д.**
		
		Ваш документ хэшируется алгоритмом md5 и полученный хэш подписывается NCAlayer-ом.

 - **Как проверяется на сервисе?**
 - **Как можно проверить подпись вне этого сервиса?**

	
## HTTP-запросы

### №1. Загрузка документа

	Загрузка документа на сервисе Учет ЭДО происходит на стороне фронта. Далее файл кодируется в base64 так же на фронте, после этого с помощью WebSocket-а подключается NCAlayer и уже непосредственно в NCAlayer отправляется строка файла хэшированная в md5 кодировке и возвращается подпись (xml строка).

### №2. Отправка подписи

	Подписывание происходит NCAlayer-ом. Сервис подключается к NCAlayer-у с помощью WebSocket-а 

	- И так чтобы подписать документ нам нужно создать подключение(connect) к webSocket-у
	- Можете попробывать через приложение webSocat или PostMan
	- В моем случае я воспользуюсь PostMan-ом
	- Итак заходим в PostMan меняем `request type` с `HTTP` на `WebSocket`
	- Вводим такой `URL`: wss://127.0.0.1:13579/ (P.S. у вас должен быть включен NCAlayer)
	- Во вкладке `Message` заполняем вот так:

  ```json
 {
	"module":"kz.gov.pki.knca.basics",
	"method":"sign",
	"args":
		{
			"allowedStorages":["PKCS12"],
			"format":"xml","data":"<root><md5-hash>..Content..</md5-hash></root>",
			"signingParams":
				{
					"decode":"false",
					"encapsulate":"true",
					"digested":"false",
					"tsaProfile":{}
				},
			"signerParams":
				{
					"extKeyUsageOids":[],
					"chain":[]
				},
			"locale":"ru"
		}
}
  ```
- `..Content..` - это хэш файла, который формируется на фронте (md5)

#### Ответ:

```json
{
    "body": {
        "result": [
            "xml подпись..."
        ]
    },
    "status": true
}
```

## Описание подписанного документа

### При подписании PDF документа

#### Сведения о документе:  <UUID_info.pdf>

Сначала идут строковые данные:
- Название подписанного документа и сервиса с помощью которого документ был подписан (*Учет.ЭДО*)
- Хеш документа (*MD5*)
- Ссылки на электронный документ отправителя и получателя

![rawData.png](https://github.com/Tr8ch/AnalysisUchet/blob/main/images/rawData.png)

Таблица:
- Время подписания документа с двух сторон
- Информация про каждого контрагента:
	- ФИО
	- Email
	- Права
	- Информация о сертификате
	- Удостоверяющий центр
	- Кем выдан
	- Номер сертификата
	- Срок сертификата

![tableData.png](https://github.com/Tr8ch/AnalysisUchet/blob/main/images/tableData.png)

Подписи отправителя и получателя:

![signature.png](https://github.com/Tr8ch/AnalysisUchet/blob/main/images/signature.png)

#### Оригинал документа:  <UUID.pdf>

Сам документ который был загружен для подписи

#### Печатная форма:  <UUID_print.pdf>

Тоже самое, только на первой странице на верхней части есть строки:
- Документ зарегистрирован и подписан с помощью сервиса Учет.ЭДО
- Ссылки на электронный документ для отправителя и получателя

![links.png](https://github.com/Tr8ch/AnalysisUchet/blob/main/images/links.png)

#### XML файл попдипсей:  <UUID.xml>

В файле лежит информация о двух подписях: отправителя и получателя.

### Для документов не PDF формата

В document.zip лежит тоже, самое что и для pdf документов за исключением одного файла:
- *Печатная форма* `UUID_print.pdf`.

> В файлах нет QR-кодов

## Описание подписей

В XML файле лежит информация о двух подписях: отправителя и получателя.

- `md5-hash` - алгоритм хеширования
- `Signature` - блок содержащит цифровую подпись, созданную с использованием стандарта XML Digital Signature
- `SignedInfo` - содержит информацию о подписываемых данных и алгоритмах, используемых для подписи
- `CanonicalizationMethod` - определяет метод каноникализации, который применяется к подписываемым данным
- `SignatureMethod` - указывает алгоритм, используемый для генерации подписи
- `Reference` - ссылка на данные, которые подписываются
- `Transforms` - информация о преобразованиях, применяемых к данным перед их подписью
- `DigestMethod` - алгоритм хеширования, используемый для создания дайджеста данных
- `DigestValue` - значение хеша, созданное на основе данных
- `SignatureValue` - значение цифровой подписи, созданное на основе данных
- `KeyInfo` - информация о ключе, использованном для создания подписи
- `X509Certificate` - сертификат содержит открытый ключ, используемый для подписи данных

В XML файле данные лежат в raw. Приведем в структуру для читабельности:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<root>
	<md5-hash>37991b585ce0f6840a3a28aa73ad3084</md5-hash>
	<ds:Signature
		xmlns:ds="http://www.w3.org/2000/09/xmldsig#">\r\n
		<ds:SignedInfo>\r\n
			<ds:CanonicalizationMethod Algorithm="http://www.w3.org/TR/2001/REC-xml-c14n-20010315"/>\r\n
			<ds:SignatureMethod Algorithm="http://www.w3.org/2001/04/xmldsig-more#rsa-sha256"/>\r\n
			<ds:Reference URI="">\r\n
				<ds:Transforms>\r\n
					<ds:Transform Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature"/>\r\n
					<ds:Transform Algorithm="http://www.w3.org/TR/2001/REC-xml-c14n-20010315#WithComments"/>\r\n
				</ds:Transforms>\r\n
				<ds:DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256"/>\r\n
				<ds:DigestValue>YtDrixblOu30TFGU/Ya+k/gn96yYN+Edh/xAnouu0yc=</ds:DigestValue>\r\n
			</ds:Reference>\r\n
		</ds:SignedInfo>\r\n
		<ds:SignatureValue>\r\nMmCvScH9Vkw6mk8IXjy4g3n4FVA41/wzH3WHkdpzE/6p4QKxULJti/u3GKXZuQAhs9C/0bI+2SjZ\r\n5wDrMH7cHwQuWtRRxxQNhRVqYN1fLhu3UbK4Ws5U+ZzGRVxg66US1Q6y82Slilox3gMbhl2pR28G\r\nQKSDxDmJWoY4dHlnleq2eUwLD8K0IoITcbI8jPkBpLAkwZVXrKJh/KMLOJb9x338iuM6K91JJ/Mr\r\nSwlZv093FKRVvONadLBFB2JaFm4+Tn8UkjAzNDMJS/CVfkel150hhq7e8pszWQrZP2cx48Em+Iwb\r\nC+H59GJUAtjOLYaGaRommJ8H7sBizWPI863saA==\r\n</ds:SignatureValue>\r\n
		<ds:KeyInfo>\r\n
			<ds:X509Data>\r\n
				<ds:X509Certificate>\r\nMIIGlTCCBH2gAwIBAgIUPuBz66CnX3d+5QhOKPrKSexZg30wDQYJKoZIhvcNAQELBQAwUjELMAkG\r\nA1UEBhMCS1oxQzBBBgNVBAMMOtKw0JvQotCi0KvSmiDQmtCj05jQm9CQ0J3QlNCr0KDQo9Co0Ksg\r\n0J7QoNCi0JDQm9Cr0pogKFJTQSkwHhcNMjMwMTE2MjE1ODM2WhcNMjQwMTE2MjE1ODM2WjCBtDEq\r\nMCgGA1UEAwwh0KHQldCd0JPQldCg0JHQkNCV0JIg0KHQkNCd0JbQkNCgMR0wGwYDVQQEDBTQodCV\r\n0J3Qk9CV0KDQkdCQ0JXQkjEYMBYGA1UEBRMPSUlOMDAwMzA3NTUwMzI1MQswCQYDVQQGEwJLWjEb\r\nMBkGA1UEKgwS0JzQo9Cl0KLQkNCg0rDQm9CrMSMwIQYJKoZIhvcNAQkBFhRzYW56aGFyXzE3MTBA\r\nbWFpbC5ydTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAIk0PrUF5JuCR9Qbq7dJU6cL\r\n0a4TcoX4nt9WuHVY9yOCmeQi74NpVqEG9pqmp+TUdvR/3F2N3xXjDetBhfCKP+K3eWDs/z43uHZt\r\nbxRr2l+RgM5NzMwWglEE1eZFxaXnotHhJiUgAJV3/GdWk8ncyvZ8IR78mucoQlLQ5XnYd/Qnm1/U\r\ntOMelMkj9CDbv8j2m4Rgis4mYEzL3ork926aQ4E+iJ3twbjt28eDGlUB4fz8lfjgb0hcIdTWWTqh\r\nIhZBbw0gPtSoB9TaxfyL4SO/yy4saXn8IHJDEAqwAzdcIhAcQVATEd2ibKn4RVG1XV5b5S8QCg1d\r\nN/YU13xpkXcrm2kCAwEAAaOCAf4wggH6MA4GA1UdDwEB/wQEAwIGwDAoBgNVHSUEITAfBggrBgEF\r\nBQcDBAYIKoMOAwMEAQEGCSqDDgMDBAMCATBeBgNVHSAEVzBVMFMGByqDDgMDAgMwSDAhBggrBgEF\r\nBQcCARYVaHR0cDovL3BraS5nb3Yua3ovY3BzMCMGCCsGAQUFBwICMBcMFWh0dHA6Ly9wa2kuZ292\r\nLmt6L2NwczBWBgNVHR8ETzBNMEugSaBHhiFodHRwOi8vY3JsLnBraS5nb3Yua3ovbmNhX3JzYS5j\r\ncmyGImh0dHA6Ly9jcmwxLnBraS5nb3Yua3ovbmNhX3JzYS5jcmwwWgYDVR0uBFMwUTBPoE2gS4Yj\r\naHR0cDovL2NybC5wa2kuZ292Lmt6L25jYV9kX3JzYS5jcmyGJGh0dHA6Ly9jcmwxLnBraS5nb3Yu\r\na3ovbmNhX2RfcnNhLmNybDBiBggrBgEFBQcBAQRWMFQwLgYIKwYBBQUHMAKGImh0dHA6Ly9wa2ku\r\nZ292Lmt6L2NlcnQvbmNhX3JzYS5jZXIwIgYIKwYBBQUHMAGGFmh0dHA6Ly9vY3NwLnBraS5nb3Yu\r\na3owHQYDVR0OBBYEFL7gc+ugp193fuUITij6yknsWYN9MA8GA1UdIwQIMAaABFtqdBEwFgYGKoMO\r\nAwMFBAwwCgYIKoMOAwMFAQEwDQYJKoZIhvcNAQELBQADggIBAEJpmfqkU7gpcKDvtyrXc4PV/BG7\r\nRFK9mJRg9TivTSMvGw2VW/s+FUbeYlfjW8PkIBaeAjPIbw2cY0yhfaHjHW0lf6B5M/E8wrl37QQ0\r\nG13jLFP5un5Fl+dQ1k6Itw5dIQ7Cn4uXxorqdDaAkbiqZRDVrzePQzsCnYAeLWj6cPzd/NZJgaUq\r\nxk5EHRuldATy/6dZyDVoQtB/TC3+BII7VwxqJxp2mwF1F/4TB1Mmq5SAp41bJFUo0bQGv77dCzUZ\r\nuCnOm0QIl9Qg0CGVC8Ju7naUnLQk4QzKAS2Fiw91D1wL7Ab5ADXAyf70xFnWlfrhX47fcYFtSngF\r\nC604EXj3VP7TtjkXxuEIppzKnphfnRy0i9+aplX88PjP4+rN7FGt6c/FSGBZ1rOVfx3O9mRvu4XV\r\nPZ/bNKo8KPcSoVC3cAorETZtZsWM9BR/dclukMBc9nH+ej4wC4hlvRhdze+WRCtZNguOvscxTa5q\r\nBg0rU3kgbQ+VHnFalvBkMe43BcrKce6GRZaSuMPa/bprTEp7k8qAqdH6EEd2mzfzBGpV2zWTWXbx\r\nexXS9shLYc7AnnVGPqlkpvLDsCY434PzFfc3LC9F5qL/91sR+RIv7mSNZQjpDwVYlLW+4pPLgWSq\r\nheR7zaRbPGz1sywnDYhThPDWN0oR3muKiCYSIOIGhAyOPnuM\r\n</ds:X509Certificate>\r\n
			</ds:X509Data>\r\n
		</ds:KeyInfo>\r\n
	</ds:Signature>
	<ds:Signature
		xmlns:ds="http://www.w3.org/2000/09/xmldsig#">\r\n
		<ds:SignedInfo>\r\n
			<ds:CanonicalizationMethod Algorithm="http://www.w3.org/TR/2001/REC-xml-c14n-20010315"/>\r\n
			<ds:SignatureMethod Algorithm="http://www.w3.org/2001/04/xmldsig-more#rsa-sha256"/>\r\n
			<ds:Reference URI="">\r\n
				<ds:Transforms>\r\n
					<ds:Transform Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature"/>\r\n
					<ds:Transform Algorithm="http://www.w3.org/TR/2001/REC-xml-c14n-20010315#WithComments"/>\r\n
				</ds:Transforms>\r\n
				<ds:DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256"/>\r\n
				<ds:DigestValue>jxb9bcCLSNvrB+ZWzE7AXYY7B6uh24u7SBc24CXc1O0=</ds:DigestValue>\r\n
			</ds:Reference>\r\n
		</ds:SignedInfo>\r\n
		<ds:SignatureValue>\r\nSeEjhcWPaw+bQklauDciz5X+vxf8ljg5QZn5/2FT/2BAtiZMguPjR87PmCbvGbpEyrpnwv8fc2Qi\r\nSrX9YxiIROr263/36ICZiMhdsKpk/n7v/TbG9ciER2L/r9p6COIphRHQ5FUY+v+aI74AByGvrBSk\r\nASvoE0w7jWBu7IlbfiCp22mvV0MHs44yQge30F6lNS4cHH03RP1Xk+bauiBs5UOuTPIpG8b2XUKx\r\nvrcILBxEGxrEB1QoM8Jp/gMpqGGs970psuxw/dKhRxyGC9J57L6iqfqEOYEy6USN5ljnQKoWfr8v\r\nHeCXOcDoyjR+ZSzNW3tQYTwD7hfxulwSnYzqZA==\r\n</ds:SignatureValue>\r\n
		<ds:KeyInfo>\r\n
			<ds:X509Data>\r\n
				<ds:X509Certificate>\r\nMIIGlTCCBH2gAwIBAgIUPuBz66CnX3d+5QhOKPrKSexZg30wDQYJKoZIhvcNAQELBQAwUjELMAkG\r\nA1UEBhMCS1oxQzBBBgNVBAMMOtKw0JvQotCi0KvSmiDQmtCj05jQm9CQ0J3QlNCr0KDQo9Co0Ksg\r\n0J7QoNCi0JDQm9Cr0pogKFJTQSkwHhcNMjMwMTE2MjE1ODM2WhcNMjQwMTE2MjE1ODM2WjCBtDEq\r\nMCgGA1UEAwwh0KHQldCd0JPQldCg0JHQkNCV0JIg0KHQkNCd0JbQkNCgMR0wGwYDVQQEDBTQodCV\r\n0J3Qk9CV0KDQkdCQ0JXQkjEYMBYGA1UEBRMPSUlOMDAwMzA3NTUwMzI1MQswCQYDVQQGEwJLWjEb\r\nMBkGA1UEKgwS0JzQo9Cl0KLQkNCg0rDQm9CrMSMwIQYJKoZIhvcNAQkBFhRzYW56aGFyXzE3MTBA\r\nbWFpbC5ydTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAIk0PrUF5JuCR9Qbq7dJU6cL\r\n0a4TcoX4nt9WuHVY9yOCmeQi74NpVqEG9pqmp+TUdvR/3F2N3xXjDetBhfCKP+K3eWDs/z43uHZt\r\nbxRr2l+RgM5NzMwWglEE1eZFxaXnotHhJiUgAJV3/GdWk8ncyvZ8IR78mucoQlLQ5XnYd/Qnm1/U\r\ntOMelMkj9CDbv8j2m4Rgis4mYEzL3ork926aQ4E+iJ3twbjt28eDGlUB4fz8lfjgb0hcIdTWWTqh\r\nIhZBbw0gPtSoB9TaxfyL4SO/yy4saXn8IHJDEAqwAzdcIhAcQVATEd2ibKn4RVG1XV5b5S8QCg1d\r\nN/YU13xpkXcrm2kCAwEAAaOCAf4wggH6MA4GA1UdDwEB/wQEAwIGwDAoBgNVHSUEITAfBggrBgEF\r\nBQcDBAYIKoMOAwMEAQEGCSqDDgMDBAMCATBeBgNVHSAEVzBVMFMGByqDDgMDAgMwSDAhBggrBgEF\r\nBQcCARYVaHR0cDovL3BraS5nb3Yua3ovY3BzMCMGCCsGAQUFBwICMBcMFWh0dHA6Ly9wa2kuZ292\r\nLmt6L2NwczBWBgNVHR8ETzBNMEugSaBHhiFodHRwOi8vY3JsLnBraS5nb3Yua3ovbmNhX3JzYS5j\r\ncmyGImh0dHA6Ly9jcmwxLnBraS5nb3Yua3ovbmNhX3JzYS5jcmwwWgYDVR0uBFMwUTBPoE2gS4Yj\r\naHR0cDovL2NybC5wa2kuZ292Lmt6L25jYV9kX3JzYS5jcmyGJGh0dHA6Ly9jcmwxLnBraS5nb3Yu\r\na3ovbmNhX2RfcnNhLmNybDBiBggrBgEFBQcBAQRWMFQwLgYIKwYBBQUHMAKGImh0dHA6Ly9wa2ku\r\nZ292Lmt6L2NlcnQvbmNhX3JzYS5jZXIwIgYIKwYBBQUHMAGGFmh0dHA6Ly9vY3NwLnBraS5nb3Yu\r\na3owHQYDVR0OBBYEFL7gc+ugp193fuUITij6yknsWYN9MA8GA1UdIwQIMAaABFtqdBEwFgYGKoMO\r\nAwMFBAwwCgYIKoMOAwMFAQEwDQYJKoZIhvcNAQELBQADggIBAEJpmfqkU7gpcKDvtyrXc4PV/BG7\r\nRFK9mJRg9TivTSMvGw2VW/s+FUbeYlfjW8PkIBaeAjPIbw2cY0yhfaHjHW0lf6B5M/E8wrl37QQ0\r\nG13jLFP5un5Fl+dQ1k6Itw5dIQ7Cn4uXxorqdDaAkbiqZRDVrzePQzsCnYAeLWj6cPzd/NZJgaUq\r\nxk5EHRuldATy/6dZyDVoQtB/TC3+BII7VwxqJxp2mwF1F/4TB1Mmq5SAp41bJFUo0bQGv77dCzUZ\r\nuCnOm0QIl9Qg0CGVC8Ju7naUnLQk4QzKAS2Fiw91D1wL7Ab5ADXAyf70xFnWlfrhX47fcYFtSngF\r\nC604EXj3VP7TtjkXxuEIppzKnphfnRy0i9+aplX88PjP4+rN7FGt6c/FSGBZ1rOVfx3O9mRvu4XV\r\nPZ/bNKo8KPcSoVC3cAorETZtZsWM9BR/dclukMBc9nH+ej4wC4hlvRhdze+WRCtZNguOvscxTa5q\r\nBg0rU3kgbQ+VHnFalvBkMe43BcrKce6GRZaSuMPa/bprTEp7k8qAqdH6EEd2mzfzBGpV2zWTWXbx\r\nexXS9shLYc7AnnVGPqlkpvLDsCY434PzFfc3LC9F5qL/91sR+RIv7mSNZQjpDwVYlLW+4pPLgWSq\r\nheR7zaRbPGz1sywnDYhThPDWN0oR3muKiCYSIOIGhAyOPnuM\r\n</ds:X509Certificate>\r\n
			</ds:X509Data>\r\n
		</ds:KeyInfo>\r\n
	</ds:Signature>
</root>
```
- В тегах <md5-hash> содержится хэш подписываемого документа. Вот небольшой пример для его получения:
  
```golang
data, _ := os.ReadFile("test.pdf")
fmt.Printf("%x\n", md5.Sum(data))
``` 

## Описание процесса верификации подписей на сервисе

> \*Данный текст не является частью анализа и нужно удалить при сдаче.
> 
> В этом разделе нужно описать процессы того, как сервис предлагает проверить подпись на легитимность. Перечислить все доступные методы рекомендованные сервисом. Возможно данная информация будет храниться в статьях, блогах или FAQ сервиса.
>  
>  Можно прилагать скрины.

## Описание процесса верификации подписей вне сервиса

> \*Данный текст не является частью анализа и нужно удалить при сдаче.
> 
> В этом разделе нужно описать процессы того, как можно проверить подписи на легитимность вне сервиса, используя технические инструменты - kalkan, openssl и т.д.

---

> \*Данный текст не является частью анализа и нужно удалить при сдаче.
> 
> Если у вас есть дополнительные секции, которые вы хотели бы включить, то оформите ее согласно структуре во всем документе.
