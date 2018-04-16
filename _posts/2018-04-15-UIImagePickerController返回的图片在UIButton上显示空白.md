---
layout: post
title:  "UIImagePickerController返回的图片在UIButton上显示空白"
date:   2018-04-15 21:10:23 -0700
categories: iOS
---

<hr>
今天遇到一个很郁闷的问题，UIImagePickerController返回的图片在UIButton上显示空白，图例如下：

![]({{ site.url }}/assets/images/1.png)

且在控制台得到error信息(这个错误其实有点误导)：

![]({{ site.url }}/assets/images/2.png)

## 思考步骤：

### 1.会不会是访问照片权限问题
因为控制台报错了，那我们就考量下是不是照片访问权限问题。
#### a)在info.plist中设置：
![]({{ site.url }}/assets/images/3.png)

#### b)检查访问照片权限：

```swift
func checkPermission() {
        let photoAuthorizationStatus = PHPhotoLibrary.authorizationStatus()
        switch photoAuthorizationStatus {
            case .authorized: print("Access is granted by user")
            case .notDetermined: PHPhotoLibrary.requestAuthorization({
            (newStatus) in
                print("status is \(newStatus)")
                if newStatus == PHAuthorizationStatus.authorized { print("success") }
        })
        case .restricted:
            print("User do not have access to photo album.")
        case .denied:
            print("User has denied the permission.")
        }
    } 
```
PHPhotoLibrary这个类在Photos中，需要导入Photos这个框架，在viewDidLoad中检查权限就可以了。但是做完权限设置控制台该报错报错，按钮图片依旧不显示。


### 2.那是不是UIImagePickerController哪里出了问题？

我这里把UIImagePickerController需要设置的地方提一下：

```swift
	//按钮点击事件
    @IBAction func handleChooseIcon() {
        let imagePickerController = UIImagePickerController()
        imagePickerController.allowsEditing = true
        imagePickerController.delegate = self
        present(imagePickerController, animated: true)
    }
```
因为我们要监听对UIImagePickerController操作的返回，所以必须设置代理，使self所指的控制器遵守UIImagePickerControllerDelegate和UINavigationControllerDelegate（系统要求的要遵守前者必须同时也遵守后者）。

然后我们看看UIImagePickerControllerDelegate方法：

```swift
@objc func imagePickerController(_ picker: UIImagePickerController, didFinishPickingMediaWithInfo info: [String : Any]) {
        if let image = info[UIImagePickerControllerEditedImage] as? UIImage {
            iconButton.setImage(image, for: .normal)
        } else if let image = info[UIImagePickerControllerOriginalImage] as? UIImage {
            iconButton.setImage(image, for: .normal)
        }
        dismiss(animated: true)
    }
```
我觉得我写的没什么大问题啊，就那么几行代码，网上说前面要加上@objc，好吧，我加。

结果令人意外地。。。。。没什么变化。问题到底出现在什么地方呢？

### 3.来看看到底UIImagePickerController有没有返回正确的image

#### a)打印返回图片尺寸

```swift
@objc func imagePickerController(_ picker: UIImagePickerController, didFinishPickingMediaWithInfo info: [String : Any]) {
        if let image = info[UIImagePickerControllerEditedImage] as? UIImage {
        	print(image.size)
            iconButton.setImage(image, for: .normal)
        } else if let image = info[UIImagePickerControllerOriginalImage] as? UIImage {           
            print(image.size)
            iconButton.setImage(image, for: .normal)
        }
        dismiss(animated: true)
    }
```

控制台结果如下：
![]({{ site.url }}/assets/images/4.png)
额。。。是有尺寸的，不过我还得确认下

#### b)打开Debug View Hierarchy
![]({{ site.url }}/assets/images/5.png)

![]({{ site.url }}/assets/images/6.png)

见鬼了！！？？居然在那里！此时模拟器上按钮的图片依旧是一片空白，我向上帝发誓，我没有看错。

### 4.UIButton的问题？
哦！猛然想起，我为了贪图系统自带的按钮点击效果把按钮的类型改为了system。那我改回去试试？

![]({{ site.url }}/assets/images/7.png)

哇靠！它出现了！

那为什么会出现这个问题呢？是渲染模式搞的鬼，其实在按钮显示的是空白图片的时候我就该发现，因为默认按钮是透明的叻
所以，其实返回的图片没有问题，是系统类型的按钮将图片渲染成了白色，这个白色取决于tint color, 如果改成红色，刚才的白色豆腐块就变成了红色了

![]({{ site.url }}/assets/images/8.png)

如果我即想要系统提供的点击效果又想显示图片，只需要把UIImagePickerController的代理方法的实现稍微改一下，把返回image的渲染模式改成不渲染。

```swift
iconButton.setImage(image.withRenderingMode(.alwaysOriginal), for: .normal)
```

大功告成！

虽然控制台依然报着那个误导我很久的错误。。。Let it go!

