# 问题场景：

稿主在使用@RequestBody注解从前端传递json数据给后端对象接收时，设置了一样的名称，但是无法java对象无法被json数据填充。
后端的自定义java对象如下所示：
![图1](web_1722930592385-1.png)

![图1](https://i-blog.csdnimg.cn/direct/122cd0d8add848b0aefb956417c58e05.png)
![图2](https://i-blog.csdnimg.cn/direct/bf09bf246f2044a887d8ec250e9be0ae.png)
前端传递的数据如下：
![图3](https://i-blog.csdnimg.cn/direct/22e33823c0374f109a18a1651c1679d9.png)
在前端json数据与后端java对象均命令为"aDn", "zDn"的情况下，后端使用@RequestBody注解时无法自动将json数据填充到java对象中。
![图4](https://i-blog.csdnimg.cn/direct/9c5eae9407b64522955d44171f0ea6b7.png)
但是神奇的是，当改变名称为“localDn", "remoteDn"后，又能够成功地接收到json数据到java对象中，如下：
![图5](https://i-blog.csdnimg.cn/direct/d3057038acfc4108b8fdcb8ff499db52.png)
![图6](https://i-blog.csdnimg.cn/direct/3c175c476f1943ba93613177b290f520.png)
![图7](https://i-blog.csdnimg.cn/direct/5c5d8beeff484c3d8e8f41df62d3aa14.png)

---

原因分析：

在更改名字后能够初步解决问题，不由地思考出现该问题的原因，经过高人指点，将问题定位到由json数据转为java对象过程中的序列化问题，在反序列化时，从java对象中找不到对应的setter方法，因此无法完成转化。通过一步步调试跟踪源码，发现在转化过程中，会使用BeanDeserializer.class这个类中的deserializerFromObject()方法完成转化，在这个方法中会先根据json数据中的字段名称来寻找java对象中的setter方法，调用找到的setter方法来向java对象中填充json数据，如下：
```java
// BeanDeserializer.class
    public Object deserializeFromObject(JsonParser p, DeserializationContext ctxt) throws IOException {
        if (this._objectIdReader != null && this._objectIdReader.maySerializeAsObject() && p.hasTokenId(5) && this._objectIdReader.isValidReferencePropertyName(p.currentName(), p)) {
            return this.deserializeFromObjectId(p, ctxt);
        } else {
            Object bean;
            if (this._nonStandardCreation) {
                if (this._unwrappedPropertyHandler != null) {
                    return this.deserializeWithUnwrapped(p, ctxt);
                } else if (this._externalTypeIdHandler != null) {
                    return this.deserializeWithExternalTypeId(p, ctxt);
                } else {
                    bean = this.deserializeFromObjectUsingNonDefault(p, ctxt);
                    return bean;
                }
            } else {
                bean = this._valueInstantiator.createUsingDefault(ctxt);
                p.setCurrentValue(bean);
                if (p.canReadObjectId()) {
                    Object id = p.getObjectId();
                    if (id != null) {
                        this._handleTypedObjectId(p, ctxt, bean, id);
                    }
                }

                if (this._injectables != null) {
                    this.injectValues(ctxt, bean);
                }

                if (this._needViewProcesing) {
                    Class<?> view = ctxt.getActiveView();
                    if (view != null) {
                        return this.deserializeWithView(p, ctxt, bean, view);
                    }
                }

                if (p.hasTokenId(5)) {
                    String propName = p.currentName();

                    do {
                        p.nextToken();
                        //在此处根据字段名寻找相应的setter方法
                        SettableBeanProperty prop = this._beanProperties.find(propName);
                        if (prop != null) {
                            try {
                                prop.deserializeAndSet(p, ctxt, bean);
                            } catch (Exception var7) {
                                Exception e = var7;
                                this.wrapAndThrow(e, bean, propName, ctxt);
                            }
                        } else {
                            this.handleUnknownVanilla(p, ctxt, bean, propName);
                        }
                    } while((propName = p.nextFieldName()) != null);
                }

                return bean;
            }
        }
    }
```
当命名为aDn, zDn时，会发现默认的setter方法为setADn, setZDn, 会自动将首字母大写，导致无法找到对应的setter方法，如下：
```java
// BeanPropertyMap.class
    public SettableBeanProperty find(String key) {
        if (key == null) {
            throw new IllegalArgumentException("Cannot pass null property name");
        } else {
            if (this._caseInsensitive) {
                key = key.toLowerCase(this._locale);
            }

            int slot = key.hashCode() & this._hashMask;
            int ix = slot << 1;
            Object match = this._hashArea[ix];
            return match != key && !key.equals(match) ? this._find2(key, slot, match) : (SettableBeanProperty)this._hashArea[ix + 1];
        }
    }
```
![图8](https://i-blog.csdnimg.cn/direct/40678c11d9ed4bea88128a419e41c9b9.png)
当调用该方法根据json中字段名寻找setter方法时，java对象中的字段名为”adn“, "zdn"，无法与json数据中的”aDn“,"zDn"相匹配，但当命名改为”localDn“,”remoteDn“时，java对象中的字段名却也为”localDn“,”remoteDn“，能够匹配找到对应的setter方法
![图9](https://i-blog.csdnimg.cn/direct/52a59544bb824dda9cc39492820565c7.png)
这是Jackson的命名策略引起的，Jackson中包含几种用于序列化时的命名策略：
```java
    public static final PropertyNamingStrategy LOWER_CAMEL_CASE = new LowerCamelCaseStrategy();
    public static final PropertyNamingStrategy UPPER_CAMEL_CASE = new UpperCamelCaseStrategy();
    public static final PropertyNamingStrategy SNAKE_CASE = new SnakeCaseStrategy();
    public static final PropertyNamingStrategy UPPER_SNAKE_CASE = new UpperSnakeCaseStrategy();
    public static final PropertyNamingStrategy LOWER_CASE = new LowerCaseStrategy();
    public static final PropertyNamingStrategy KEBAB_CASE = new KebabCaseStrategy();
    public static final PropertyNamingStrategy LOWER_DOT_CASE = new LowerDotCaseStrategy();
```
这些策略都大都是根据属性名中的单词来分隔的，而”aDn,zDn“在命名时转为了”adn,zdn"，但是“localDn, remoteDn”却能保持不变，因些在命名时要遵循相应的规范，不能随意起取“aDn, zDn”这种驼峰首单词只有一个字母的情况的属性名。

---

# 解决方案：
再使用@RequesBody从json格式转化为java对象时，可以在java对象中使用@JsonProperty注解，写明对象中属性值与json中名称的对应关系，这样能够使得在进行json序列化与反序列化时能够正确匹配。
![图10](https://i-blog.csdnimg.cn/direct/341cda6e4495474bbd76fce41e86034d.png)
