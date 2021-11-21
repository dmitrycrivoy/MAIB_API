# MAIB API

## Предисловие
После обращения в банк вам должны дать ссылку на ресурсы, которые включают в себя инструкции по установке и тестированию, API, правила использования и другую информацию. 
Здесь сделаем упор именно на процесс тестирования, с которым могут возникнуть сложности (у меня возникли), но также рассмоттрим другую полезную информацию.
Оригинал информации о настройке и тестировании (на румынском языке):
<https://docs.google.com/document/d/1ZEkmYhlfHQs9VRHIENbHcVwLMvCLCH5vb-hYHvaVP4s/edit>


## Подготовка
### Сертификаты
У вас должен быть файл в pfx (pkcs12) формате, который включает в себя закрытый ключ, сертификат удостоверяющего центра (CA - Certification Authority) и .
Он необходим для доступа к тестовому или продакшн серверу (в зависимотси от ваших нужд вам выдадут соответствующий).
Но для дальнейшей работы с этими сертификатами и ключом их необходимо извлечь из pfx-файла.

#### Извлечение сертификатов из pfx-файла
Сохранять будем с расширением pem, как это изначально преполагается для использования.
```sh
# Создать CA (Certification Authority) сертификат:
openssl pkcs12 -in <filename>.pfx -out cacert.pem -cacerts -nokeys

# Создать сам сертификат:
openssl pkcs12 -in <filename>.pfx -out pcert.pem -clcerts -nokeys

# Создать приватный ключ с парольной фразой (без неё никак не экпортировать ключ из pfx):
openssl pkcs12 -in <filename>.pfx -out key.key -nocerts

# Затем создать приватный ключ без парольной фразы:
openssl rsa -in key.key -out key.pem

```
Необходим приватный ключ именно без парольной фразы, потому что иначе работать не будет.  
Проверить ключ и сертификат можно с помощью хешей:
```sh
openssl x509 -noout -modulus -in cert.pem | openssl md5
openssl rsa -noout -modulus -in key.pem | openssl md5
```
Оба хеша должны быть одинаковыми.

### Адреса серверов
Адреса тестовых серверов:  
Merchant: https://ecomm.maib.md:4499/ecomm2/MerchantHandler  
Client:   https://ecomm.maib.md:7443/ecomm2/ClientHandler

Адреса продакшн серверов:  
Merchant: https://maib.ecommerce.md:11440/ecomm01/MerchantHandler  
Client:   https://maib.ecommerce.md:443/ecomm01/ClientHandler


## Тестирование
После подтверждения IP вашего сервера как доверенного, банк обяжет вас пройти проверку команд.
Для данной проверки можно использовать код, указанный ниже. Также среди файлов от MAIB возможно будут html-формы, которые также можно использовать для проверки.
Я использовал первый вариант, так как он немного быстрее в реализации.

На данный момент банк описывает следующую последовательность проверки команд (лучше уточнить у банка):
1. Генерация платежа (команда = V).
2. Проверка статуса платежа (команда = C).
3. Процедура полного/частичного возврата платежа (команда = R).
4. Закрытие рабочего дня (команда = B).

После генерации платежа (шаг 1) будет выдан TRANSACTION_ID.
Он понадобится для прохождения проверки 3DS тестовой карты в форме платежа (то есть как проверка оплаты созданного платежа с точки зрения пользователя).

**Тестовая карта:**  
```
Номер карты: 5102180060101124  
Срок действия: 06/28  
CVV: 760
```

Затем его нужно будет использовать как один из параметров в функциях кода (php) или как trans_id формы html для проверки следующих по списку команд.

**Какая именно функция (и последовательность её параметров) в коде соответствует вышеуказанной команде можно уточнить в файле _MaibClient.php_.**

Пояснения для некоторых полей:  
base_url - адрес тестового или продакшн сервера (Merchant).  
verify - полученный ранее сертификат cacert.pem.  
cert - полученный ранее сертификат pcert.pem.  
ssl_key - полученный ранее ключ key.pem.

```php
namespace MyProject;
require_once(__DIR__ . '/vendor/autoload.php');

use Fruitware\MaibApi\MaibClient;
use Fruitware\MaibApi\MaibDescription;
use GuzzleHttp\Client;
use GuzzleHttp\Subscriber\Log\Formatter;
use GuzzleHttp\Subscriber\Log\LogSubscriber;
use Monolog\Handler\StreamHandler;
use Monolog\Logger;

//set options
$options = [
	'base_url' => 'https://ecomm.maib.md:4499/ecomm2/MerchantHandler',
	'debug'  => true,
	'verify' => false,
	'defaults' => [
		'verify' => __DIR__.'/cert/cacert.pem',
		'cert'    => [__DIR__.'/cert/pcert.pem', 'Pem_pass'],
		'ssl_key' => __DIR__.'/cert/key.pem',
		'config'  => [
			'curl'  =>  [
				CURLOPT_SSL_VERIFYHOST => false,
				CURLOPT_SSL_VERIFYPEER => false,
			]
		]
	],
];

// init Client
$guzzleClient = new Client($options);

// create a log for client class, if you want (monolog/monolog required)
$log = new Logger('maib_guzzle_request');
$log->pushHandler(new StreamHandler(__DIR__.'/logs/maib_guzzle_request.log', Logger::DEBUG));
$subscriber = new LogSubscriber($log, Formatter::SHORT);

$client = new MaibClient($guzzleClient);
$client->getHttpClient()->getEmitter()->attach($subscriber);
// examples

//register sms transaction
var_dump($client->registerSmsTransaction('1', 978, '127.0.0.1', '', 'ru'));

//register dms authorization
var_dump($client->registerDmsAuthorization('1', 978, '127.0.0.1', '', 'ru'));

//execute dms transaction
var_dump($client->makeDMSTrans('1', '1', 978, '127.0.0.1', '', 'ru'));

//get transaction result
var_dump($client->getTransactionResult('1', '127.0.0.1'));

//revert transaction
var_dump($client->revertTransaction('1', '1'));

//close business day
var_dump($client->closeDay());

```

html-форма для генерации платежа:
```html
<form action="https://ecomm.maib.md:4499/ecomm2/MerchantHandler" method="post">
	<input type="text" name="command" value="V" /><br>
	<input type="text" name="amount" value="100" /><br>
	<input type="text" name="client_ip_addr" value="127.0.0.1" /><br>
	<input type="text" name="currency" value="498" /><br>
	<input type="text" name="language" value="ru" /><br>
	<input type="text" name="description" value="Product/Service Description" /><br>
	<input type="submit" value="Submit" name="Submit" /><br>
</form>
```

html-форма для проверки платежа:
```html
<form action="https://ecomm.maib.md:4499/ecomm2/MerchantHandler" method="post">
	<input type="text" name="command" value="C" /><br>
	<input type="text" name="trans_id" value="TRANS_ID" /><br>
	<input type="submit" value="Submit" name="Submit" /><br>
</form>
```
