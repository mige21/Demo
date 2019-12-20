
# CoDi Mobile Wrapper


El objetivo de este wrapper es manejar toda la lógica, así como peticiones y manejo de datos que se requieran para usar el servicio de CoDi, con el fin de que cualquier App pueda implementarlo de forma sencilla.



## Tabla de contenido
- __[Codi Wrapper](#codi\ wrapper)__
  + [Inicialización](#inicialización) 
  + [Status codi](#status-codi) 
  + [Set Password](#set-password) 
  + [Enrolamiento inical](#enrolamiento-inical) 
  + [Obtener Google ID](#obtener-google-id) 
  + [Actualización](#actualización) 
  + [Aplicación por default](#aplicación-por-default) 
  + [Validar Cuenta](#validar-cuenta) 
  + [Estatus de cuenta](#estatus-de-cuenta) 
  + [Generar Mensaje de Cobro](#generar-mensaje-de-cobro) 
  + [Consulta mensaje de cobro](#consulta-mensaje-de-cobro) 
  + [Resolver mensaje de cobro (consulta claves de descifrado)](#resolver-mensaje-de-cobro--consulta-claves-de-descifrado--) 
  + [Descifrar Mensaje de Cobro no presencial (Notificación Push)](#descifrar-mensaje-de-cobro-no-presencial--notificación-push--)
  + [Baja de CoDi](#baja-de-codi) 
- **[Api Manager](#api\ manager)**
- Secure Helper
- Storage Manager
- Push Helper

## CoDi Wrapper 
Es el encargado en tener toda la interacción con el <span style="color:#d73a49">App Móvil</span>, se encarga de gestionar y orquestar todas las llamadas que se tengan que hacer a los módulos de bajo nivel.

### Métodos:

#### <span style="color:#6f42c1">Inicialización</span>
Inicializa la instancia del CoDiWrapper, será llamado al inicializar la app y guardará en la instancia los datos del <span style="color:#d73a49">certificactionId</span> y el <span style="color:#d73a49">certificationString</span>.

```kotlin
    CoDiWrapper.getInstance(context: Context, certificationId: String, certificationString: String)
```

Donde:
| Data | Descripción
| --- | --- |
| certificationId | Identificador de certificación asignado por Banco de México a cada participante. |
| certificationString | Cadena de certificación asignada por Banco de México a cada participante. |

#### <span style="color:#6f42c1">Status codi</span> 
Regresará si CoDi está activo o no.

```kotlin
    CoDiWrapper.isActive(): boolean
```

#### <span style="color:#6f42c1">Set Password</span> 
Se hace uso de este pwd en el StorageManager, con el fin de utilizarlo como llave para recuperar y almacenar información de manera segura, se debe de mandar a llamar esta función cada vez que el app ha terminado de cargar.

```kotlin
    CoDiWrapper.setPwd(pass: String)
```

Donde:
| Data | Descripción
| --- | --- |
| pass | puede ser el nombre de usuario o cualquier otra cadena. |

#### <span style="color:#6f42c1">Enrolamiento inical</span> 
Pedirá el número de teléfono celular para inicializar el enrolamiento a CoDi.

```kotlin

    CoDiWrapper.initEnrollment(phoneNumber: String, Callback {
        fun onSucces() {
            //enrolamiento correcto
        }
        fun onError(errorStatus) {
        
        } 
    })

```

Internamente el CodiWrapper calculará los datos neceserios para generar la petición y solicitar al <span style="color:#d73a49">ApiManager</span> el registro inical.

Datos a calcular:
- Epoch -> Fecha actual en milisegundos
- Hash -> Resultado del Sha512 de la concatenación del <span style="color:#d73a49">epoch</span> y el <span style="color:#d73a49">certificationString</span>

Al responder correctamente el <span style="color:#d73a49">ApiManager</span> deberá guardar en la misma instancia el GoogleID (<span style="color:#d73a49">gId</span>)
y el VerificationCode (<span style="color:#d73a49">dv</span>) devuelto por el método y avisar que fue correcto el enrolamiento inicial.

#### <span style="color:#6f42c1">Obtener Google ID</span>
Pedirá el OTP que llega por SMS y regresará el Google Id ya desencriptado, deberá guardar el codigo de verificación, el Google Id y el KeySouce en el dispositivo usando el StorageManager.

```kotlin
    CoDiWrapper.getGoogleId(verificationOTP: String): String
```

#### <span style="color:#6f42c1">Actualización</span> 
Se mandará llamar después de la inicializaciíon de CoDi, como cada vez que se haga Login, servirá para actualizar el dato del FirebaseId y notificará si la <span style="color:#d73a49">App Móvil</span> está marcada como default.

```kotlin

    CoDiWrapper.update(firebaseId: String, Callback {
        fun onSucces({isDefaultApp: boolean}) {
            //actualización completa
        }
        fun onError(errorStatus) {
        } 
    })

```
Mandará llamar al <span style="color:#d73a49">ApiManager</span> para completar el registro subsequente

Datos a calcular:
- Hmac: Usando el <span style="color:#d73a49">SecureHelper</span>

```kotlin
    var hmac = SecureHelper.hmacSha256(keySource, (nc + dv + IdN) )
    // el dv es completado a 3 dígitos
```

Al momento de responder correctamente este método se deberá Guardar el número celular y guardará una bandera para saber que ya fue activado CoDi.

Se deberá guardar el keysourceLocal encriptado de la siguiente forma

```kotlin
    var keySourceLocal = Sha512Hex(pwd+keySource)
```

Donde el **`pwd`** es el username del cliente

#### <span style="color:#6f42c1">Aplicación por default</span> 
Ayuda a definir si es que se quiere usar esta app por default para operaciones con CoDi.

```kotlin
    CoDiWrapper.setDefaultApp(Callback {
        fun onSucces() {
            //la app ahora es la App x default
        }
        fun onError(errorStatus) {
        
        } 
    })
```

La función anterior manda llamar al <span style="color:#d73a49">ApiManager</span> que se encargará de registrar el app por default, si la respuesta del servicio es correcta se guarda en el StorageManager el valor de la app por omision.

#### <span style="color:#6f42c1">Validar Cuenta</span>
Posterior al enrolamiento se tiene que mandar a validar una cuenta la cual será asociada al servicio de CoDi.

```kotlin
   CoDiWrapper.accountsValidate(account: String, type: Enum, Callback {
        fun onSucces(trackingCode: String) {
        
        }
        fun onError(errorStatus) {
        
        } 
    })
```

Donde:
| Data | Descripción
| --- | --- |
| account | CLABE, no de tarjeta de débito, número celular |
| type | <span style="color:#e36209">CLABE, DEBIT_CARD o PHONE_NUMBER</span> |
| trackingCode | código de rastreo regresado por Banxico |

Se solicita al <span style="color:#d73a49">ApiManager</span> realice la validación de la cuenta, en caso de ser exitosa regresará un código de rastreo, este se guardará junto con el estatus de validación en el Storage Manager.

#### <span style="color:#6f42c1">Estatus de cuenta</span>
En caso de no recibir la notificación push de validación o interumpir el flujo, se hará uso de esta función para revisar el estatus de una cuenta.

```kotlin

    CoDiWrapper.checkAccountStatus(trackingCode: String, Callback {
        fun onSucces(status: Enum) {
        
        }
        fun onError(errorStatus) {
        
        } 
    })

```

Donde:

| Data | Descripción
| --- | --- |
| trackingCode | código de rastreo devuelto por Banxico |
| status | <span style="color:#e36209">PENDING, VERIFIED, ERROR</span> |

El <span style="color:#d73a49">ApiManager</span> consultará el estatus de la validación, si este es `VERIFIED` quiere decir que la cuenta fue validada correctamente, se guardará el status en el Storage Manager.

#### <span style="color:#6f42c1">Generar Mensaje de Cobro</span> 
Genera el string necesario para compartir un mensaje de cobro, para ello recibe un `PaymentMessage` que contendrá los datos del mensaje de cobro y regresará el mensaje de cobro encriptado.

  + __Nota:__ La App deberá permitir capturar el concepto y opcionalmente el monto y la referencia numérica.

```kotlin

    CoDiWrapper.generatePaymentMessage(paymentMessage: PaymentMessage, serial: Int, acct: String, name: String): PaymentMessageEncrypted {
        ...
        return payMessage
    })

```

Internamente el CodiWrapper genera un mensaje de cobro cifrado.

Donde:

| Data | Descripción
| --- | --- |
| paymentMessage | objeto de tipo PaymentMessage|
| serial | Número de serie único de cobro|
| acct | Cuenta configurada en la app para la recepción de pagos.|
| name | Nombre del vendedor o beneficiario|
| PaymentMessageEncrypted | mensaje de cobro generado y cifrado por el CodiWrapper |

#### <span style="color:#6f42c1">Consulta mensaje de cobro</span>

Consulta directamente el estado de las solicitudes de cobro, ya sean las enviadas por un vendedor o las recibidas por un comprador.

```kotlin

    CoDiWrapper.consultPaymentMessages(vendorPhone:String, vDv:String, idC: String, buyerPhone: String?, buyerDv: Int?, numberPage: Int, 
        Callback {
            fun onSucces(paymentMessages: Array) {
            
            }
            fun onError(errorStatus) {
            
            }
    })

```

El <span style="color:#d73a49">ApiManager</span> se encargará de hacer la consulta y el resultado de esta será una lista con todos los mensajes de cobro procesados por los distintos compradores.

Donde:

| Data | Descripción
| --- | --- |
| vendorPhone | Alias del número de celular beneficiario o del número de serie del certificado. |
| vDv | Dígito verificador del dispositivo del vendedor. |
| idC | Identificador del Mensaje de Cobro generado del lado del vendedor. |
| buyerPhone |  Alias del Número de Celular Ordenante (opcional). |
| buyerDv | Digito Verificador del Dispositivo Ordenante. |
| numberPage | Número de página a consultar. |
| paymentMessages | Mensajes de cobro desencriptados. |

Posteriormente el `SecureHelper` hace el descifrado de la lista de mensajes de cobro.

#### <span style="color:#6f42c1">Resolver mensaje de cobro (consulta claves de descifrado)</span> 
Desencriptará el mensaje de cobro y lo devuelve en formato `PaymentMessage`.

```kotlin

    CoDiWrapper.decryptPaymentMessage(jsonPaymentMessage: String, Callback {
        fun onSucces(paymentMessage: PaymentMessage) {
        
        }
        fun onError(errorStatus) {
        
        } 
    })

```

El <span style="color:#d73a49">ApiManager</span> hará la consulta para poder recuperar las llaves de descifrado, una vez recuperadas se descifra el mensaje y se valida la información de cobro para verificar la integridad.

#### <span style="color:#6f42c1">Descifrar Mensaje de Cobro no presencial (Notificación Push)</span>

Banco de México enviará dicho Mensaje de Cobro, a través del Servidor de Notificaciones, al dispositivo del cliente.

```kotlin

    CodiWrapper.decryptSendedMessage(idC: String, serial: Long, messageEncrypted: String): PaymentMessage? {
        ...
        return paymentMessage
    }

```

El SecureHelper se encarga de hacer el descifrado del mensaje y el  <span style="color:#d73a49">ApiManager</span> crea un objeto de tipo `PaymentMessage` a partir del mensaje descifrado.

Donde:

| Data | Descripción
| --- | --- |
| idC | Identificador del mensaje de cobro asignado por Banco de México. |
| serial | Valor numérico asignado por Banco de México.  |
| messageEncrypted | Información cifrada del mensaje de cobro. |
| paymentMessage |  objeto de tipo PaymentMessage devuelto por el ApiManager  |

#### <span style="color:#6f42c1">Baja de Codi</span> 
Borra de StorageManager los datos almacenados durante el proceso de enrolamiento CoDi.

```kotlin

    CodiWrapper.logoutCodi()

```

## Api Manager

El api manager es el encargado de interactuar via internet con el API de CoDi

### Servicios

#### <span style="color:#28a745">initEnrollment</span> 
Servicio de enrolamiento inicial.

```js

    var apiManager = ApiManager()
    apimanager.initEnrollment(phoneNumber: String, certificationId: String, certificationString: String, 
        Callback {
            onSuccess({maskedGoogleId: String, verificationCode: String }) {
            
            }
            onError(errorStatus: Int) {
            
            }
    })

```


#### <span style="color:#28a745">subEnrollment</span> 
Servicio de enrolamiento subsecuente.

```js

    var apiManager = ApiManager()
    apimanager.subEnrollment(phoneNumber: String, firebaseId: String, verificationCode: String, hmac: String, 
        Callback {
            onSuccess({defaultVerificationCode: String}) {
            
            }
            onError(errorStatus: Int) {
            
            }
    })

```



- <span style="color:#28a745">setDefaultApp:</span> Servicio para settear la `App móvil` como app por omisión de CoDi

```js

    var apiManager = ApiManager()
    apimanager.setDefaultApp(phoneNumber String, hmac:String, Callback {
        onSuccess() {
        
        }
        onError(errorStatus: Int) {
        
        } 
    })

```

- <span style="color:#28a745">validateAccount:</span> Servicio para validar cuenta

```js
    var apiManager = ApiManager()
    apimanager.validateAccount(phoneNumber String, account: String, type: Enum, hmac: String, Callback {
        onSuccess({trackingCode: String}) {
        
        }
        onError(errorStatus: Int) {
        
        }
    })

```



- <span style="color:#28a745">getAccountStatus:</span> Servicio para validar cuenta

```js

    var apiManager = ApiManager()
    apimanager.getAccountStatus(phoneNumber: String, trackingCode: String, hmac: String, Callback {
        onSuccess({cypheredAccountInfo: String}) {
        
        }
        onError(errorStatus: Int) {
        
        }
    })

```

- <span style="color:#28a745">getStatusPayment:</span> Servicio para revisar Mensajes de Cobro

```js

    var apiManager = ApiManager()
    apimanager.getStatusPayment(idMessage: String, payee: Payee, payer: Payer, length:
    Int, page: Int, hmac: String ,Callback {
        onSuccess({cypheredList: String, isLastPage: boolean}) {
        
        }
        onError(errorStatus: Int) {
        
        }
    })

```

- <span style="color:#28a745">getDecryptKeys:</span> Consultar la llave de descifrado del mensaje de cobro

```js

    var apiManager = ApiManager()
    apimanager.getDecryptKeys(idMessage: String, payee: Payee, payer: Payer, serial: String, cypheredPaymentMessage: String, hmac: String ,Callback {
        onSuccess({cypheredList: String, isLastPage: boolean}) {
        
        }
        onError(errorStatus: Int) {
        
        }
    })

```


## SecureHelper

Su función es manejar todo el tema de encripción, desencripción y encoding o decoding de datos.

### Métodos:

- <span style="color:#ffd33d">Sha512</span> -> Hash string to 512, recibe como parámetro el string a realizar hash
- <span style="color:#ffd33d">Sha256</span> -> Hash string to 256, recibe como parámetro el string a realizar hash
- <span style="color:#ffd33d">hmacSha256</span> -> Calcula el hmac en Sha256, recibe como parámetro el key y el string a encodear.

