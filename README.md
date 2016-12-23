Yii2 Oauth2 Server
==================

A wrapper for implementing an OAuth2 Server(https://github.com/bshaffer/oauth2-server-php).
This is a fork of https://github.com/Filsh/yii2-oauth2-server. I decided to fork it after a long standing issue of branches not being resolved at original repo.
There is no guarantee that the two will remain compartible so check out things to see if there is anything broke while transiting.

Installation
------------

The preferred way to install this extension is through [composer](http://getcomposer.org/download/).

Either run

```
php composer.phar require --prefer-dist hosannahighertech/yii2-oauth2-server "*"
```

or add

```json
"hosannahighertech/yii2-oauth2-server": "dev-master"
```

to the require section of your composer.json. 

To use this extension,  simply add the following code in your application configuration:

```php
'bootstrap' => ['oauth2'],
'modules' => [
    'oauth2' => [
        'class' => 'filsh\yii2\oauth2server\Module',
        'tokenParamName' => 'accessToken',
        'tokenAccessLifetime' => 3600 * 24,
        'storageMap' => [
            'user_credentials' => 'common\models\User',
        ],
        'grantTypes' => [
            'user_credentials' => [
                'class' => 'OAuth2\GrantType\UserCredentials',
            ],
            'refresh_token' => [
                'class' => 'OAuth2\GrantType\RefreshToken',
                'always_issue_new_refresh_token' => true
            ]
        ]
    ]
]
```

### Configure User class

```common\models\User``` - user model implementing an interface ```\OAuth2\Storage\UserCredentialsInterface```, so the oauth2 credentials data stored in user table. Make sure you implement:

1.  `public function checkUserCredentials($username, $password)` which Checks the supplied username and password for validity and returns TRUE if the username and password are valid, and FALSE if it isn't.

2. `public function getUserDetails($username)`  which returns array of the associated "user_id" and optional "scope" values something like this: 

```php
[ 
    "user_id"  => USER_ID,    // REQUIRED user_id to be stored with the authorization code or access token
    "scope"    => SCOPE       // OPTIONAL space-separated list of restricted scopes
] 

```

Additional OAuth2 Flags:

`enforceState` - Flag that switch that state controller should allow to use "state" param in the "Authorization Code" Grant Type

`allowImplicit` - Flag that switch that controller should allow the "implicit" grant type

The next step your shold run migration

```php
 yii migrate --migrationPath=@vendor/hosannahighertech/yii2-oauth2-server/migrations
```

this migration create the oauth2 database scheme and insert test user credentials ```testclient:testpass``` for ```http://fake/```

add url rule to urlManager

```php
'urlManager' => [
    'rules' => [
        'POST oauth2/<action:\w+>' => 'oauth2/rest/<action>',
        ...
    ]
]
```

Usage
-----

To use this extension,  simply add the behaviors for your base controller:

```php
use yii\helpers\ArrayHelper;
use yii\filters\auth\HttpBearerAuth;
use yii\filters\auth\QueryParamAuth;
use filsh\yii2\oauth2server\filters\ErrorToExceptionFilter;
use filsh\yii2\oauth2server\filters\auth\CompositeAuth;

class Controller extends \yii\rest\Controller
{
    /**
     * @inheritdoc
     */
    public function behaviors()
    {
        return ArrayHelper::merge(parent::behaviors(), [
            'authenticator' => [
                'class' => CompositeAuth::className(),
                'authMethods' => [
                    ['class' => HttpBearerAuth::className()],
                    ['class' => QueryParamAuth::className(), 'tokenParam' => 'accessToken'],
                ]
            ],
            'exceptionFilter' => [
                'class' => ErrorToExceptionFilter::className()
            ],
        ]);
    }
}
```

Create action authorize in site controller for Authorization Code

`https://api.mysite.com/authorize?response_type=code&client_id=TestClient&redirect_uri=https://fake/`

```php
/**
 * SiteController
 */
class SiteController extends Controller
{
    /**
     * @return mixed
     */
    public function actionAuthorize()
    {
        if (Yii::$app->getUser()->getIsGuest())
            return $this->redirect('login');
    
        /** @var $module \filsh\yii2\oauth2server\Module */
        $module = Yii::$app->getModule('oauth2');
        $response = $module->handleAuthorizeRequest(!Yii::$app->getUser()->getIsGuest(), Yii::$app->getUser()->getId());
    
        /** @var object $response \OAuth2\Response */
        Yii::$app->getResponse()->format = \yii\web\Response::FORMAT_JSON;
    
        return $response->getParameters();
    }
}
```

For More on Requests and responses as well as explanations for Oauth Grant types see excellet tutorial by Jenkov here: http://tutorials.jenkov.com/oauth2/

[see more](http://bshaffer.github.io/oauth2-server-php-docs/grant-types/authorization-code/)

Also if you set ```allowImplicit => true```  you can use Implicit Grant Type - [see more](http://bshaffer.github.io/oauth2-server-php-docs/grant-types/implicit/)

Request example:

`https://api.mysite.com/authorize?response_type=token&client_id=TestClient&redirect_uri=https://fake/cb`

With redirect response:

`https://fake/cb#access_token=2YotnFZFEjr1zCsicMWpAA&state=xyz&token_type=bearer&expires_in=3600`

### JWT Tokens
If you want to get Json Web Token (JWT) instead of convetional token, you will need to set `'useJwtToken' => true` in module and then define two more configurations: 
`'public_key' => 'app\storage\PublicKeyStorage'` which is the class that implements [PublickKeyInterface](https://github.com/bshaffer/oauth2-server-php/blob/develop/src/OAuth2/Storage/PublicKeyInterface.php) and `'access_token' => 'OAuth2\Storage\JwtAccessToken'` which implements [JwtAccessTokenInterface.php](https://github.com/bshaffer/oauth2-server-php/blob/develop/src/OAuth2/Storage/JwtAccessTokenInterface.php). The Oauth2 base library provides the default [access_token](https://github.com/bshaffer/oauth2-server-php/blob/develop/src/OAuth2/Storage/JwtAccessToken.php) which works great. Just use it and everything will be fine.

Here is a sample class for public key implementing `PublickKeyInterface`. Make sure that paths to private and public keys are correct. You can generate them with OpenSSL tool with two steps (Thanks to (Rietta)[https://rietta.com/blog/2012/01/27/openssl-generating-rsa-key-from-command/]):

1. ```openssl genrsa -des3 -out privkey.pem 2048```

2. ```openssl rsa -in privkey.pem -outform PEM -pubout -out pubkey.pem```

**Note that** you can copy contents of the files and paste the long strings in class variables. It is nasty but it works fine if that is what you want.

```php
<?php
namespace app\security\storage;

class PublicKeyStorage implements \OAuth2\Storage\PublicKeyInterface{


    private $pbk =  null;
    private $pvk =  null; 

    public function __construct()
    {
        $pvkText =  file_get_contents(dirname(__FILE__).'/../keys/privkey.pem');        
        $this->pvk = openssl_get_privatekey($pvkText, 'YOUR_PASSPHRASE_IF_ANY');
        
        $this->pbk =  file_get_contents(dirname(__FILE__).'/../keys/pubkey.pem'); 
    }

    public function getPublicKey($client_id = null){ 
        return  $this->pbk;
    }

    public function getPrivateKey($client_id = null){
        return  $this->pvk;
    }

    public function getEncryptionAlgorithm($client_id = null){
        return "RS256";
    }

}

``` 


For more, see https://github.com/bshaffer/oauth2-server-php
