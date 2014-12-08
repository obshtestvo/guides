# Подкарване на HTTPS


## Безплатен сертификат от https://startssl.com

Безплатните сертификати се издават за top level domain + един subdomain. Поради тази причина например gov.obshtestvo.bg сертификата е валиден за obshtesvo.bg и gov.obshtevo.bg, но не и www.gov.obshtestvo.bg . Ако е необходим wildcard сертификат трябва да се купува.

За да се активират другите domain-и трябва да се получи 1 писмо на hostmaster@domain (може postmaster@) от certmaster@startcom.org с auth code. Заради spam filter-a не винаги тръгва от първия път. 

Самата регистрация и последващото подписване на сертификата отнема няколко часа, защото са се валидира от хора.

Процеса по регистрация в startssl и издаването на сертификат е описан подробно с картинки в https://github.com/ioerror/duraconf/blob/master/startssl/README.markdown (ние ползваме малко по-силни сертификати от описаните там).


## startssl за проектите на Общество

Ние имаме startssl.com account с поща ssl@obshtestvo , която препраща към @tochev и @mitio . И двамата имат сертификат за достъп. Account-a е на името на mitio.

Желателно е да се използва този account, с цел да не се забравя да се подновят сертификати и избягването на single point of failure.


## Издаване на сертификат

```
mkdir /etc/certificates/FOO
cd /etc/certificates/FOO
openssl req -new -nodes -sha256 -keyout FOO.key -out FOO.csr -newkey rsa:4096
# с това казваме 4096bit RSA, с sha256 fingerprint (sha1 се счита за deprecated)
# Тук е важно да се сложи правилно "Common Name" на съответния domain и email
# Реално startssl игнорира всичко, но ако се подписва от друг не е така
chmod o-r FOO.key

# .csr се подписва от избрания provider (примерно startssl)
# слагаме резултата в FOO.crt (с нов ред накрая)

# правене на chain за nginx, за другите сървъри е малко по-различно
cat FOO.crt  ../obshtestvo.bg/startssl.certchain > FOO.chained.crt
```

където `FOO` е domain-a


## Настройване на ngiпx

 * Правим `FOO.chained.crt` (описано в последната команда на предходната част)
 * Модифицираме `/etc/nginx/sites-available/FOO` базирано на `/etc/nginx/sites-available/gov`
   * Може да се добави сертификата и в guest-a, но е желателно да се избягва за `foo.obshtestvo.bg`, т.к. това значи работещите по този проект да имат сертификат за `obshtestvo.bg`
 * Конфигурацията ни е базирана на https://github.com/ioerror/duraconf/blob/master/configs/nginx/nginx.conf (там има примери и за други сървъри)
 * TODO: forwarding protocol


### Важни неща за текущата web server конфигурацията

 * на текущата конфигурация на server-a на общество https default-a е obshtestvo.bg, което значи че browser-и без [SNI](http://en.wikipedia.org/wiki/Server_Name_Indication) ще виждат този сертификат
 * сложено е задължително [forward secrecy](https://en.wikipedia.org/wiki/Forward_secrecy) , което **не се поддържа от IE<=8 на XP**
 * изключен е SSL3, което *е проблем за IE<=6 на XP* (но може да се оправи от потребителя и доста банки са изпратили писма как се оправя)
 * по принцип е хубаво http://foo permanently да redirect-ва към https
 * включен е ssl caching (note: при SNI setup той трябва да е включен за всички site-ове или изключен)
 * [strict transport security](https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security) (казва на browser-a да достъпва този site само по https) е сложено на 7 дена, recommended minimum-a е 180 дена, но това значи commitment да има има https (също така трябва да се види дали да включва subdomains - по-добре не)
 * ssl конфигурацията е взаимствана от https://github.com/ioerror/duraconf/blob/master/configs/nginx/nginx.conf , което значи че определени шифри са разрешени, а не както преди - определени шифри да са забранени


## Тестване

 * отваря се https сайта и се гледа да няма "alert"-и в address bar-a
   * най-честия проблем са неща, които са през http
 * Пускане на domain-a в https://www.ssllabs.com - имат много добра документация
   * пример  https://www.ssllabs.com/ssltest/analyze.html?d=obshtestvo.bg


## Често срещани проблеми

 * Достъп до ресурси по http (js libs, CDN css/images) - оправя се като се направята url-ите на protocol relative - от `http://foo/bar` на `//foo/bar`
 * Hardcode-нати url-и на site-a за emails/rss/etc.
 * TODO: more


