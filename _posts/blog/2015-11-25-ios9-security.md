---
layout: post
title: iOS9安全特性概述(更新SE的部分)
description: 谈谈刚刚释出的iOS9在安全上的一些新特性
category: blog
---

iOS9已经发布了几个月了，近期研究了下其中几个安全特性，作下总结吧。

## Keychain
使用过iOS或者OS X系统的同学可能对Keychain不会陌生，按照官方解释，这是一个存储敏感个人信息的地方，如密钥、密钥、证书等等。实际上Keychain本质上就是一个sqlite数据库。研究过越狱机的同学会知道，keychain的路径就是

	/private/var/keychains/keychain-2.db
Keychain安全性很大程度来源于沙盒机制，第三方App只能通过系统API陷入内核进行keychain操作，而不能直接使用该路径下的keychain数据库，系统API可以做比较全面的访问控制，从而达到安全性要求。

iOS9中对Keychain又做了若干安全方面的改进。
<img src="/images/blog/iOS9/keychain.png">

* 首先，keychain中敏感数据的物理存储位置已经迁到Secure Enclave(iOS设备上自5s开始增加的一个协处理器)中。
* Keychain的访问属性**kSecAttrAccessibleAlways**将被deprecated掉。
* 当使用TouchID来保护Keychain中的数据时，可以通过**touchIDAuthenticationAllowableReuseDuration**来指定TouchID验证的间隔，间隔期之内可以免验证读取Keychain数据。

## Application Password
Keychain中的元素现在可以通过手机passcode和app级的密码双重加密。这样即使手机解锁，app中数据仍然无法访问，对keychain元素的访问控制粒度更加细化。
<img src="/images/blog/iOS9/app_password.png">

	 CFErrorRef error = NULL;
	 //ACL
	 SecAccessControlRef sacObject = SecAccessControlCreateWithFlags(kCFAllocatorDefault,
	                        kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
	                        kSecAccessControlTouchIDAny | kSecAccessControlApplicationPassword,
	                        &error);
	
	 LAContext *context = [[LAContext alloc] init];
	 NSString *password = @"e693b64e405e9ddc578959b97665e750";
	  //设置Application Password   
	 [context setCredential:[password dataUsingEncoding:NSUTF8StringEncoding] type:LACredentialTypeApplicationPassword];
	
	 NSData *secretPasswordTextData = [@"SECRET_PASSWORD_TEXT" dataUsingEncoding:NSUTF8StringEncoding];
	
	 NSDictionary *attributes = @{
	                (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
	                (__bridge id)kSecAttrService: @"SampleService",
	                (__bridge id)kSecValueData: secretPasswordTextData,
	                (__bridge id)kSecUseNoAuthenticationUI: @YES,
	                (__bridge id)kSecAttrAccessControl: (__bridge_transfer id)sacObject,
	                (__bridge id)kSecUseAuthenticationContext: context
	            };
	
	 OSStatus status = SecItemAdd((__bridge CFDictionaryRef)attributes, nil);
这样就添加了一个需要同时解锁手机和提供Application Password才能访问的keychain item.

## Secure Enclave
这是一个在iPhone5s上开始引入的模块，最初被设计用来存放指纹等敏感信息(TouchID), 用户录入的指纹被提取关键特征后存放在SE中，之后用户通过TouchID验证时都会将当前的特征送到SE中，与录入的特征做对比，返回对比结果:YES or NO.自始自终，用户的指纹都不会离开SE.

苹果自iOS8开始开放TouchID给第三方App使用。而到了iOS9,SE也终于可以供第三方App使用了。虽然只有简单的3个API.个人认为却是安全软件最注重的一个特性。

* SecGenerateKeyPair.该方法用于生成RSA或者ECDSA非对称密钥。iOS9中可以指定参数，似的密钥直接在SE中生成，并且私钥直接存入SE，公钥返回给调用方。任何用户都无法获取到私钥，只能通过SecKeyRawSign方法请求签名。对应的用SecKeyRawVerify来进行签名的验证。一个简短的例子说明一下


		CFErrorRef error = NULL;
		Sec AccessControlRef sacObject;
		//ACL
		sacObject = SecAccessControlCreateWithFlags(kCFAllocatorDefault,
		                                            kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
		                                            kSecAccessControlTouchIDAny | kSecAccessControlPrivateKeyUsage, &error);
		
		// 密钥参数
		NSDictionary *parameters = @{
		    (__bridge id)kSecAttrTokenID: (__bridge id)kSecAttrTokenIDSecureEnclave, //私钥存入SE
		    (__bridge id)kSecAttrKeyType: (__bridge id)kSecAttrKeyTypeEC, //如果要使用SE，这里只能指定kSecAttrKeyTypeEC
		    (__bridge id)kSecAttrKeySizeInBits: @256,
		    (__bridge id)kSecPrivateKeyAttrs: @{
		        (__bridge id)kSecAttrAccessControl: (__bridge_transfer id)sacObject,
		        (__bridge id)kSecAttrIsPermanent: @YES,
		        (__bridge id)kSecAttrLabel: @"my-se-key", //密钥名
		    },
		};
		
		//产生公私钥对
		SecKeyRef publicKey, privateKey;
		OSStatus status = SecKeyGeneratePair((__bridge CFDictionaryRef)parameters, &publicKey, &privateKey);
		//将公钥保存下来

这里返回的publicKey为实际的公钥数据，而privateKey仅仅为私钥在SE中对应的token,通过该token只能请求签名，而无法dump出私钥的具体内容。App可以在完成上述调用后将公钥保存到服务端,以备后面进行签名验证。

* 接下来要做的是将公钥上传到服务端做保存，以便在后续验证的时候使用。可惜这个api设计的及其讨厌，SecGenerateKeyPair返回的既非公钥数据也非标准格式，而仅仅是一个类似句柄的东西。如果想得到具体的公钥，还需要一点技巧。

		NSDictionary *pubDict = @{
		        (__bridge id)kSecClass              : (__bridge id)kSecClassKey,
		        (__bridge id)kSecAttrKeyType        : (__bridge id)kSecAttrKeyTypeEC,
		        (__bridge id)kSecAttrLabel          : @"",
		        (__bridge id)kSecAttrIsPermanent    : @(YES),
		        (__bridge id)kSecValueRef           : (__bridge id)publicKey,
		        (__bridge id)kSecAttrKeyClass       : (__bridge id)kSecAttrKeyClassPublic,
		        (__bridge id)kSecReturnData         : @(YES)
		    };
		
		    CFTypeRef dataRef = NULL;
		    status = SecItemAdd((__bridge CFDictionaryRef)pubDict, &dataRef);

通过将publicKey添加到Keychain并同时得到返回值的方式来拿到具体的公钥数据dataRef。现在已经得到了具体的公钥了，然而这是一个NSData，想要转换成标准的pem格式还需要继续努力

* 上面得到的是一个公钥的裸数据，确实必要的头信息，所以首先我们需要附上必要的头信息。

		#define kCryptoExportImportManagerSecp256r1CurveLen     256
		unsigned char kCryptoExportImportManagerSecp256r1header[] =
		    {   0x30, 0x59, 0x30, 0x13, 0x06, 0x07,
		        0x2A, 0x86, 0x48, 0xCE, 0x3D, 0x02,
		        0x01, 0x06, 0x08, 0x2A, 0x86, 0x48,
		        0xCE, 0x3D, 0x03, 0x01, 0x07, 0x03,
		        0x42, 0x00
		    };
		
		#define kCryptoExportImportManagerSecp256r1headerLen    26
		
		NSMutableData *data = [NSMutableData dataWithBytes:kCryptoExportImportManagerSecp256r1header
		                                                length:sizeof(kCryptoExportImportManagerSecp256r1header)];
		[data appendData:raw_data]; //data为公钥裸数据
  
 
接着需要对该数据做一下base64然后附上头尾信息就ok了。

	#define kCryptoExportImportManagerPublicKeyInitialTag       "-----BEGIN PUBLIC KEY-----\n"
	#define kCryptoExportImportManagerPublicKeyFinalTag         "-----END PUBLIC KEY-----"
	#define kCryptoExportImportManagerPublicNumberOfCharactersInALine   64
	
	pem_pub_key = kCryptoExportImportManagerPublicKeyInitialTag;
	NSString *base64String = [data base64EncodedStringWithOptions:NSDataBase64Encoding64CharacterLineLength];
	pem_pub_key += __STDSTRING(base64String);
	pem_pub_key += "\n";
	pem_pub_key += kCryptoExportImportManagerPublicKeyFinalTag;

好了，一个标准的pem格式的公钥完成了。形状就是长成这样的：

	-----BEGIN PUBLIC KEY-----
	MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEKvkwKGqmPfg6FyeTIX4/L+rNBpo6
	fJQbUbNJCTmw9MIgSL8Kxn9pi+rH9P34LFnH6vvwfLTtNeYRmoILtNp+wg==
	-----END PUBLIC KEY----- 
 
接下来就是请求签名了

	NSDictionary *query = @{
	        (__bridge id)kSecClass: (__bridge id)kSecClassKey,
	        (__bridge id)kSecAttrKeyClass: (__bridge id)kSecAttrKeyClassPrivate,
	        (__bridge id)kSecAttrLabel: @"my-se-key", //生成密钥的名字
	        (__bridge id)kSecReturnRef: @YES,
	        (__bridge id)kSecUseOperationPrompt: @"Authenticate to sign data" //TouchID验证的title文案
	    };
	
	SecKeyRef privateKey;
	//获取私钥token
	OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, (CFTypeRef *)&privateKey); 、
	
	uint8_t signature[128];
	size_t signatureLength = sizeof(signature);
	uint8_t digestData[32];
	size_t digestLength = sizeof(digestData);
	status = SecKeyRawSign(privateKey, kSecPaddingPKCS1, digestData, digestLength, signature, &signatureLength);
 

比较巧妙的是，由于生成密钥时的ACL中指定了**kSecAccessControlTouchIDAny**,这样在请求签名时候，会自动弹出TouchID的验证界面，从而指纹的验证和签名的生成作为一个原子操作都被封装到SE中，外界无法伪造指纹的通过来骗取签名，这是一个非常重要的安全特性。

到这里我们就生成了一个签名，通常的流程是将该签名发到服务端去验证，查看签名是否正确。
需要注意的是，在SE中生成ECC密钥时，SE中将只保存私钥，并不保存公钥，所以这里我们如果想通过指定kSecAttrKeyClass为public来查找公钥是无法奏效的。App自己负责保存公钥。

然后使用我们上面导出的pem格式公钥来完成签名的验证试试:（使用openssl）

	BIO *buf = BIO_new_mem_buf((void*)pemPubKey.c_str(), (int)pemPubKey.size());
	EC_KEY *ecKey = PEM_read_bio_EC_PUBKEY(buf, NULL, NULL, NULL);
	EC_KEY_print_fp(stdout, ecKey, 2);
	
	uint8_t hash[CC_SHA256_DIGEST_LENGTH];
	CC_SHA256(plain.UTF8String, (CC_LONG)plain.length, hash);
	
	int ret = ECDSA_verify(0, hash, sizeof(hash), &signature[0], signatureLength, ecKey);
 
ret为1表示验签通过，噢耶! 中间的公钥输出是这样的:

	Private-Key: (256 bit)
	  pub: 
	      04:2a:f9:30:28:6a:a6:3d:f8:3a:17:27:93:21:7e:
	      3f:2f:ea:cd:06:9a:3a:7c:94:1b:51:b3:49:09:39:
	      b0:f4:c2:20:48:bf:0a:c6:7f:69:8b:ea:c7:f4:fd:
	      f8:2c:59:c7:ea:fb:f0:7c:b4:ed:35:e6:11:9a:82:
	      0b:b4:da:7e:c2
	  ASN1 OID: prime256v1
	  NIST CURVE: P-256

好，到此完毕。

<div id="ckepop">
<span class="jiathis_txt">分享到：</span>
<a class="jiathis_button_tsina">新浪微博</a>
<a class="jiathis_button_weixin">微信</a>

<a href="http://www.jiathis.com/share" class="jiathis jiathis_txt jiathis_separator jtico jtico_jiathis" target="_blank">更多</a>
<a class="jiathis_counter_style"></a>
</div>
<script type="text/javascript" src="http://v2.jiathis.com/code/jia.js" charset="utf-8"></script>
