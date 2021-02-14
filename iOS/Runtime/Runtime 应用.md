# Runtime 应用
#iOS知识点/Runtime

# API
![](Runtime%20%E5%BA%94%E7%94%A8/3DAE79FB-003D-405E-AC1D-45B5FB87421A.png)
![](Runtime%20%E5%BA%94%E7%94%A8/45A5F039-F87F-4B34-8A78-EDAA85FB55AB.png)
![](Runtime%20%E5%BA%94%E7%94%A8/387AB703-1325-428F-B63D-FFAD0FE11BF0.png)
![](Runtime%20%E5%BA%94%E7%94%A8/6F616477-D3A0-46D4-A18F-F60BCEDF0CCA.png)

# 字典和模型的转换
Runtime 遍历ivar_list ,结合KVC赋值，可能用到的API：
	* 使用 class_copyIvarList() 函数获取成员变量的列表
	* 使用 class_copyPropertyList() 函数获取属性列表
	* 通过 property_getAttributes() 方法获取属性的参数。
	* ivar_getTypeEncoding() 函数可以获取到属性的类型。
	* method_getTypeEncoding() 函数可以获取到方法类型编码
``` objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
     Class cls = Person.class;
     unsigned int count = 0;
    
     Person * person = [[Person alloc] init];
     NSDictionary * dict = @{  @"name" : @"Tom", @"age" : @19, @"height": @175 };
    
     Ivar * ivars = class_copyIvarList(cls, &count);
    
     for (int i = 0; i < count; i++) {
          const char * clsName = ivar_getName(ivars[i]);
          NSString * name = [NSString stringWithUTF8String:clsName];
          NSString * key = [name substringFromIndex:1];  // 去掉'_'
          [person setValue:dict[key] forKey:key];
     }
}

2018-11-04 19:42:16.964474+0800 Demo[6425:1574210] height:175.0000，name:Tom，age:19，time:(null)
```

# NSCoding 归档和解档
和模型转换类似，基于KVC完成编码和解码。
``` objc
- (void)encodeWithCoder:(NSCoder *)aCoder
{
     unsigned int count = 0;
     Ivar * ivars = class_copyIvarList(self.class, &count);

     for (int i = 0; i < count; i++) {
          const char * ivarName = ivar_getName(ivars[i]);
          NSString * name = [NSString stringWithUTF8String:ivarName];
          NSString * key  = [name substringFromIndex:1];
        
          id value = [self valueForKey:key];  // 取出 key 对应的 value
          [aCoder encodeObject:value forKey:key];   // 编码
     }
}

- (id)initWithCoder:(NSCoder *)aDecoder
{
     if (self = [super init]) {

          unsigned int count = 0;
          Ivar * ivars = class_copyIvarList(self.class, &count);

          for (int i = 0; i < count; i++) {
               const char * ivarName = ivar_getName(ivars[i]);
               NSString * name = [NSString stringWithUTF8String:ivarName];
               NSString * key = [name substringFromIndex:1];
            
               id value = [aDecoder decodeObjectForKey:key];  // 解码
               [self setValue:value forKey:key];  // 设置 key 对应的 value
          }
     }
     return self;    
}
```

# 方法交换
拦截按钮事件，
![](Runtime%20%E5%BA%94%E7%94%A8/E73942A5-FE34-4200-A4E1-8D6CB8820222.png)
```swift
protocol SelfAware: class {
    static func awake()
    static func swizzlingForClass(_ forClass: AnyClass, originalSelector: Selector, swizzledSelector: Selector)
}

extension SelfAware {
    
    static func swizzlingForClass(_ forClass: AnyClass, originalSelector: Selector, swizzledSelector: Selector) {
        let originalMethod = class_getInstanceMethod(forClass, originalSelector)
        let swizzledMethod = class_getInstanceMethod(forClass, swizzledSelector)
        guard (originalMethod != nil && swizzledMethod != nil) else {
            return
        }
        if class_addMethod(forClass, originalSelector, method_getImplementation(swizzledMethod!), method_getTypeEncoding(swizzledMethod!)) {
            class_replaceMethod(forClass, swizzledSelector, method_getImplementation(originalMethod!), method_getTypeEncoding(originalMethod!))
        } else {
            method_exchangeImplementations(originalMethod!, swizzledMethod!)
        }
    }
}

class NothingToSeeHere {
    static func harmlessFunction() {
        let typeCount = Int(objc_getClassList(nil, 0))
        let types = UnsafeMutablePointer<AnyClass>.allocate(capacity: typeCount)
        let autoreleasingTypes = AutoreleasingUnsafeMutablePointer<AnyClass>(types)
        objc_getClassList(autoreleasingTypes, Int32(typeCount))
        for index in 0 ..< typeCount {
            (types[index] as? SelfAware.Type)?.awake()
        }
        types.deallocate()
    }
}
extension UIApplication {
    private static let runOnce: Void = {
        NothingToSeeHere.harmlessFunction()
    }()
    override open var next: UIResponder? {
        UIApplication.runOnce
        return super.next
    }
}
```
1. 首先可以定义一个协议提供一个swizzling交换方法，让遵循的类去自己实现交换逻辑。
2. 重写UIApplication Next属性（applicationDidFinishLaunching之前调用），在返回前通过Runtime ClassList方法遍历所有遵循了协议的类并调用swizzling交换方法。
3. 继承协议的类实现swizzling交换方法，主要通过runtime的exchangeImplementations方法。

> 交换系统方法时需要注意为了不避免意向不到的情况。在新方法里，最后再调用一次该新方法（此时已经过替换，此方法就是原来的方法）  

**注意：**
平常应该慎用runtime的一些黑魔法，导致一些安全问题。
[iOS 界的毒瘤：Method Swizzle - iOS - 掘金](https://juejin.im/entry/5a1fceddf265da43310d9985)
换推荐使用： [RSSwizzle](https://github.com/rabovik/RSSwizzle/) 

# 动态添加方法，拦截未实现的方法
继承自 NSObject 的类有两个类方法，一个适用于类方法，一个适用于对象方法。代码中调用没有实现的方法时，也就是 sel 标识的方法没有实现，都会先调用这两个方法中的一个拦截。那么可以重写下面的方法，并在内部通过class_addMethod动态的添加方法。
``` objc
+ (BOOL)resolveClassMethod:(SEL)sel;
+ (BOOL)resolveInstanceMethod:(SEL)sel;
```
``` objc
+ (BOOL)resolveInstanceMethod:(SEL)sel
{
     if ([NSStringFromSelector(sel) isEqualToString:@"doSomething"]) {       
         //返回值：YES if the method was found and added to the receiver, otherwise NO.
         class_addMethod(self, sel, method, "v@:");  // 为 sel 指定实现为 method
     }
     return YES;
}
```
