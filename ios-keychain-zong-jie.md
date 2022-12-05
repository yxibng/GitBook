# iOS Keychain 总结

## Keychain 是什么

一个加密的数据库， 可以用来存储小的用户数据， 包括

* kSecClassGenericPassword：通用密码（可以用来存自定义数据）
* kSecClassInternetPassword：互联网密码
* kSecClassCertificate：证书
* kSecClassKey：秘钥
* kSecClassIdentity：证书+秘钥

![](https://fastly.jsdelivr.net/gh/yxibng/filebed@main/img/images/blog/16692950150511669295014987.png) Figure 1 Securing the user's secrets in a keychain

打开 mac 系统的 keychain 观察一下， 发现系统有多个keychain，每个keychain都有上图对应的项目 ![](https://fastly.jsdelivr.net/gh/yxibng/filebed@main/img/images/blog/16694375672261669437566402.png)

## 如何管理数据，增删查改

![](https://fastly.jsdelivr.net/gh/yxibng/filebed@main/img/images/blog/16692963470741669296346746.png) Figure 1 Putting data and attributes into a keychain

keychain 将要存储的数据封装为 SecKeychainItemRef，通过存取 SecKeychainItem 来达到对数据的管理。

SecKeychainItemRef 包含

* data， CFData， 要存取的数据
* attributes, 该条目的属性，存取权限，查询标签等

提供的api

```
//添加
OSStatus SecItemAdd(CFDictionaryRef attributes, CFTypeRef  _Nullable *result);
//查询
OSStatus SecItemCopyMatching(CFDictionaryRef query, CFTypeRef  _Nullable *result);
//更新
OSStatus SecItemUpdate(CFDictionaryRef query, CFDictionaryRef attributesToUpdate);
//删除
OSStatus SecItemDelete(CFDictionaryRef query);

```

### SecItemAdd

```
OSStatus SecItemAdd(CFDictionaryRef attributes, CFTypeRef  _Nullable *result);
```

#### 参数

**attributes**

一个字典，描述要添加的数据：

* 要添加数据类型，对应的 key 为`kSecClass`, value 可以是
  * kSecClassGenericPassword：通用密码（可以用来存自定义数据）
  * kSecClassInternetPassword：互联网密码
  * kSecClassCertificate：证书
  * kSecClassKey：秘钥
  * kSecClassIdentity：证书+秘钥
* 数添加的数据本身, key 为 `kSecValueData`, value 为 `CFDataRef` 类型
* 关联的属性, 参考：[Item Attribute Keys and Values](https://developer.apple.com/documentation/security/keychain\_services/keychain\_items/item\_attribute\_keys\_and\_values?language=objc)
  * 每种类型关联的属性可以不同，例如 kSecClassInternetPassword 可以关联 `kSecAttrServer`, `kSecAttrPort`, `kSecAttrProtocol`， 而 `kSecClassGenericPassword`就没有这些选项
  * `kSecAttrAccessGroup` 可以指定存入的keychain group，这个会影响到查询范围
* 添加后，返回添加结果的类型，指定类型的数据会存入 result 字段。参考：[ Item Return Result Keys](https://developer.apple.com/documentation/security/keychain\_services/keychain\_items/item\_return\_result\_keys?language=objc)

**result**

添加结束，函数返回，指向新添加的条目。根据 attributes 中指定的类型，返回对应的值。参考： [Item Return Result Keys](https://developer.apple.com/documentation/security/keychain\_services/keychain\_items/item\_return\_result\_keys?language=objc)

一般添加不关心新条目，传nil

#### 返回值

成功或者错误码。参考：[Security Framework Result Codes](https://developer.apple.com/documentation/security/1542001-security\_framework\_result\_codes?language=objc)

#### 注意

1. kSecAttrAccessGroup 可以指定添加到 钥匙串分组，该分组中的成员共享该条目
2. 一次添加多条，可以使用`kSecUseItemList` 传入字典数组，但是不能用于密码类型
3. 添加是同步执行，放到主线程可能会卡界面，可以用异步方式

```
- (void)addKeychainItemWithAttributes:(CFDictionaryRef)attrs completion:(void(^)(OSStatus status, CFTypeRef item))completion {
    dispatch_async(backgroundQueue, ^{
        CFTypeRef item = NULL;
        OSStatus addResult = SecItemAdd(attrs, &item);
        completion(addResult, item);
        if (item) {
            CFRelease(item);
        }
    });
}
```

### SecItemCopyMatching

```
OSStatus SecItemCopyMatching(CFDictionaryRef query, CFTypeRef  _Nullable *result);
```

通过 query 指定查询条件，将查询结果存入 result 中。

#### 参数

**query** 用于指定查询条件

* 查询的数据类型，通过 `kSecClass` 指定
* 关联的属性， 用于缩小查询范围
* 查询参数，返回一条还是多条匹配项，是否大消息敏感，或者从特定项中搜索。参考： [Search Attribute Keys and Values](https://developer.apple.com/documentation/security/keychain\_services/keychain\_items/search\_attribute\_keys\_and\_values?language=objc)
* 指定返回类型，指定的类型不同，result 中返回的类型就不同。 参考： [Item Return Result Keys](https://developer.apple.com/documentation/security/keychain\_services/keychain\_items/item\_return\_result\_keys?language=objc)

\*\*result \*\*

保存查询结果。 例如： 指定 `kSecReturnData: kCFBooleanTrue` ，result 中存的就是`CFDataRef` 类型的数据。参考： [Item Return Result Keys](https://developer.apple.com/documentation/security/keychain\_services/keychain\_items/item\_return\_result\_keys?language=objc)

#### 返回值

成功或者错误码。参考：[Security Framework Result Codes](https://developer.apple.com/documentation/security/1542001-security\_framework\_result\_codes?language=objc)

#### 注意

1. 默认返回第一个匹配项，如果想查询多条匹配，指定 `kSecMatchLimit` 大于1， 此时 result 为 `CFArrayRef` 类型
2. 当类型为`kSecInternetPasswordItemClass` 或 `kSecGenericPasswordItemClass`时， 不能同时指定`kSecReturnData` 和 `kSecMatchLimitAll`, 原因是拷贝每个密码时，需要额外的授权验证。
3. 默认情况下，会搜索当前keychain 以及所有加入的 keychain group， 通过 `kSecAttrAccessGroup` 可以指定要在哪个 keychain group 查询
4. 主线程同步执行会卡UI， 可以异步执行

```
- (void)findKeychainItemWithAttributes:(NSDictionary *)attributes completion:(void(^)(OSStatus status, CFTypeRef item))completion {
    dispatch_async(backgroundQueue, ^{
        CFDictionaryRef attrs = (__bridge CFDictionaryRef)attributes;
        CFTypeRef item = NULL;
        OSStatus result = SecItemCopyMatching(attrs, &item);
        completion(result, item);
        CFRelease(item);
    });
}
```

### SecItemUpdate

```
OSStatus SecItemUpdate(CFDictionaryRef query, CFDictionaryRef attributesToUpdate);
```

#### 参数

**query**

用来指定要更新的条目, 执行更新，需要先查询到要更新的条目。 参考 SecItemCopyMatching 如何指定查询条件。

**attributesToUpdate**

指定需要更新的属性。

#### 返回值

成功或者错误码。参考：[Security Framework Result Codes](https://developer.apple.com/documentation/security/1542001-security\_framework\_result\_codes?language=objc)

#### 注意

主线程同步执行会卡UI， 可以异步执行

```
- (void)updateKeyChainItemWithAttributes:(CFDictionaryRef)attrs update:(CFDictionaryRef)update completion:(void(^)(OSStatus status))completion {
    dispatch_async(backgroundQueue, ^{
        OSStatus updateResult = SecItemUpdate(attrs, update);
        completion(updateResult);
    });
}
```

### SecItemDelete

```
OSStatus SecItemDelete(CFDictionaryRef query);
```

#### 参数

**query**

用来指定要删除的的条目， 参考SecItemCopyMatching 如何指定查询条件。

#### 返回值

成功或者错误码。参考：[Security Framework Result Codes](https://developer.apple.com/documentation/security/1542001-security\_framework\_result\_codes?language=objc)

#### 注意

1. 默认删除所有的匹配项，
2. 可以通过指定`kSecMatchItemList` 来限定删除范围， `kSecMatchItemList`对应的值的类型为`CFArrayRef` , 数组中每一项的类型必须相同，
   * 针对临时引用，每一项是一个引用，引用的类型可以是
     * SecKeychainItemRef,
     * SecKeyRef
     * SecCertificateRef
     * SecIdentityRef
     * CFDataRef
   * 针对持久化引用, 引用的必须是类型是 `CFDataRef`
3. 主线程同步执行会卡UI， 可以异步执行

```
- (void)deleteKeychainItemWithAttributes:(CFDictionaryRef)attrs completion:(void(^)(OSStatus status))completion {
    dispatch_async(backgroundQueue, ^{
        OSStatus deleteResult = SecItemDelete(attrs);
        completion(deleteResult);
    });
}
```

#### 什么是临时引用，持久化引用又是什么？

**临时引用**

对应的 key 值是`kSecReturnRef`

> The corresponding value is of type `CFBooleanRef`. A value of `kCFBooleanTrue` indicates that a reference should be returned. Depending on the item class requested, the returned references may be of type `SecKeychainItemRef`, `SecKeyRef`, `SecCertificateRef`, `SecIdentityRef`, or `CFDataRef`.

示例 ：SecCertificateRef 证书对象的引用在keychain中存取

创建SecCertificateRef

```
SecCertificateRef certificate =
    SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certData);
		 
if (certificate)  { CFRelease(certificate); } // After you are done with it
```

将证书存入keychain

```
NSDictionary* addquery = @{ (id)kSecValueRef:   (__bridge id)certificate,
                            (id)kSecClass:      (id)kSecClassCertificate,
                            (id)kSecAttrLabel:  @"My Certificate",
                           };
						   
						   
OSStatus status = SecItemAdd((__bridge CFDictionaryRef)addquery, NULL);
if (status != errSecSuccess) {
    // Handle the error
}						   
```

从keychain读取证书

```
NSDictionary *getquery = @{ (id)kSecClass:     (id)kSecClassCertificate,
                            (id)kSecAttrLabel: @"My Certificate",
                            (id)kSecReturnRef: @YES,
                            };
SecCertificateRef certificate = NULL;
OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)getquery,
                                      (CFTypeRef *)&certificate);
if (status != errSecSuccess) { <# Handle error #> }
else                         { <# Use certificate #> }
 
if (certificate) { CFRelease(certificate); } // After you are done with it
```

观察到这里用的就是 `kSecReturnRef`, 代表临时引用。

* SecKeychainItemRef,
* SecKeyRef
* SecCertificateRef
* SecIdentityRef
* CFDataRef

**持久化引用**

key 为 kSecReturnPersistentRef。

> The corresponding value is of type `CFBooleanRef`. A value of kCFBooleanTrue indicates that a persistent reference to an item should be returned as a `CFDataRef` object. Unlike normal references, a persistent reference may be stored on disk or passed between processes.

1. 一个持久化引用，引用的对象必须是`CFDataRef`
2. 持久化引用可以存储在磁盘，可以跨进程传递

## 示例

### [Adding a Password to the Keychain](https://developer.apple.com/documentation/security/keychain\_services/keychain\_items/adding\_a\_password\_to\_the\_keychain?language=objc)

```
struct Credentials {
    var username: String
    var password: String
}

enum KeychainError: Error {
    case noPassword
    case unexpectedPasswordData
    case unhandledError(status: OSStatus)
}

static let server = "www.example.com"

let account = credentials.username
let password = credentials.password.data(using: String.Encoding.utf8)!
var query: [String: Any] = [kSecClass as String: kSecClassInternetPassword,
                            kSecAttrAccount as String: account,
                            kSecAttrServer as String: server,
                            kSecValueData as String: password]
let status = SecItemAdd(query as CFDictionary, nil)
guard status == errSecSuccess else { throw KeychainError.unhandledError(status: status) }

```

### [Searching for Keychain Items](https://developer.apple.com/documentation/security/keychain\_services/keychain\_items/searching\_for\_keychain\_items?language=objc)

```
let query: [String: Any] = [kSecClass as String: kSecClassInternetPassword,
                            kSecAttrServer as String: server,
                            kSecMatchLimit as String: kSecMatchLimitOne,
                            kSecReturnAttributes as String: true,
                            kSecReturnData as String: true]

var item: CFTypeRef?
let status = SecItemCopyMatching(query as CFDictionary, &item)
guard status != errSecItemNotFound else { throw KeychainError.noPassword }
guard status == errSecSuccess else { throw KeychainError.unhandledError(status: status) }

guard let existingItem = item as? [String : Any],
    let passwordData = existingItem[kSecValueData as String] as? Data,
    let password = String(data: passwordData, encoding: String.Encoding.utf8),
    let account = existingItem[kSecAttrAccount as String] as? String
else {
    throw KeychainError.unexpectedPasswordData
}
let credentials = Credentials(username: account, password: password)

```

### [Updating and Deleting Keychain Items](https://developer.apple.com/documentation/security/keychain\_services/keychain\_items/updating\_and\_deleting\_keychain\_items?language=objc)

```
let query: [String: Any] = [kSecClass as String: kSecClassInternetPassword,
                            kSecAttrServer as String: server]
							
let account = credentials.username
let password = credentials.password.data(using: String.Encoding.utf8)!
let attributes: [String: Any] = [kSecAttrAccount as String: account,
                                 kSecValueData as String: password]

//update								 								 
let status = SecItemUpdate(query as CFDictionary, attributes as CFDictionary)
guard status != errSecItemNotFound else { throw KeychainError.noPassword }
guard status == errSecSuccess else { throw KeychainError.unhandledError(status: status) }	

//delete
let status = SecItemDelete(query as CFDictionary)
guard status == errSecSuccess || status == errSecItemNotFound else { throw KeychainError.unhandledError(status: status) }
```

## keychain group & app group

参考：[Sharing Access to Keychain Items Among a Collection of Apps](https://developer.apple.com/documentation/security/keychain\_services/keychain\_items/sharing\_access\_to\_keychain\_items\_among\_a\_collection\_of\_apps?language=objc)

> Enable apps to share keychain items with each other by adding the apps to an access group.

通过keychain group 可以在 app 间共享keychain item。

> An access group is a logical collection of apps tagged with a particular group name string. Any app in a given group can share keychain items with all the other apps in the same group. You can add an app to any number of groups, but the app is always part of at least one group that contains only itself. That is, an app can always store and retrieve private keychain items, regardless of whether it also participates in any other groups. Keychain items, on the other hand, are always part of exactly one group.

默认情况下，app 加入了以自身id 为标记的分组中，这个分组中的所有 keychain item 仅限app 自身访问。

通过加入其他分组，app 间可以通过 keychain group 共享 keychain item。

### keychain group 类型

You control the groups that your app belongs to by manipulating its entitlements. In particular, an app belongs to all the groups named in a virtual array of strings that the system forms for each app as the concatenation of the following items, evaluated in this order:

**Keychain access groups** The optional Keychain Access Groups Entitlement holds an array of strings, each of which names an access group.

显示指定加入的 keychain group， group id为 `[$(teamID).$(keychain group id)]`

**Application identifier** Xcode automatically adds the application-identifier entitlement (or the com.apple.application-identifier entitlement in macOS) to every app during code signing, formed as the team identifier (team ID) plus the bundle identifier (bundle ID).

隐式的加入了以自身为组的 keychain group, group id为 `[$(teamID).$(bundle ID)]`

**Application groups** When you collect related apps into an application group using the App Groups Entitlement, they share access to a group container, and gain the ability to message each other in certain ways. Starting in iOS 8, the array of strings given by this entitlement also extends the list of keychain access groups.

由于加入了app group， 隐式的加入了以 app group 为组的 keychain group, group id为 `[$(app group id)]`

Xcode handles the application identifier (app ID) for you when you set the bundle ID. You set the others by manipulating capabilities in Xcode.

添加keychain group 和 app group 可以在xcode 中通过操作capabilities来实现。 ![](https://fastly.jsdelivr.net/gh/yxibng/filebed@main/img/images/blog/16694538450341669453844996.png) ![](https://fastly.jsdelivr.net/gh/yxibng/filebed@main/img/images/blog/16694538730181669453872352.png)

### app group 和 keychain group 区别

App groups and keychain access groups aren’t mutually exclusive—you can use both in the same app—but they do differ in several important ways that may help you decide which to use for a given situation.

两者不是互相排斥的， 一个app可以同时加入keychain group 和 app group。

First, as described above, using an app group enables additional data sharing beyond keychain items. You might want this extra sharing, or might already be using an app group for this purpose, and thus not need to add keychain access groups. On the other hand, you might not want to enable this additional sharing at all, and prefer keychain access groups instead.

app group 除了共享 keychain item 之外，还能共享其他内容，例如共享偏好设置

```
let userDefualts = UserDefaults.init(suiteName: "$(group_Id)")
userDefualts?.setValue(data, forKey: key)
userDefualts?.synchronize()
```

Second, order matters. The system considers the first item in the list of access groups to be the app’s default access group. This is the access group that keychain services assumes if you don’t otherwise specify one when adding keychain items. An app group can’t ever be the default, because the app ID is always present and appears earlier in the list. However, a keychain access group can be the default, because it appears before the app ID. In particular, the first keychain access group, if any, that you specify in the corresponding capability becomes the app’s default access group. If you don’t specify any keychain access groups, then the app ID is the default.

关于默认 group， 当添加，查询，更新，删除 keychain item 时，不指定`kSecAttrAccessGroup` 时，默认操作的是哪个group ？

1. 系统以 group list 数组中的第一个为默认 group
2. 默认group 不会是app group， 因为它不会是第一个
3. 如果加入keychain group， 该 keychain group 可能为默认的 keychain group
4. 想限定keychain item 只在 app 内访问，不能被keychain group 访问，需要将 bundle id 所在的keychain group 置顶，或者明确指定使用 bundle id 标记的keychain group
