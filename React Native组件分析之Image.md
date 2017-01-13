#React Native组件分析之Image

> 本文基于React Native 0.32 对 官方提供的`Image`组件进行分析。

`Image`是一个用于显示多种图片类型的React组件，它可以显示来自网络，assets目录，本地SD卡，用户自定义目录的图片。官网给出的用法如下：
```javascript
renderImages() {
  return (
    <View>
      <Image
        style={styles.icon}
        source={require('./icon.png')}
      />
      <Image
        style={styles.logo}
        source={{uri: 'http://facebook.github.io/react/img/logo_og.png'}}
      />
    </View>
  );
}
```
那么它是如何实现的呢？为了更好地对RN如何封装一个自定义组件进行说明，下面我将分别从JS端和Native端的源码进行分析。


----
##React端
 在JS端，Image组件的源码位于
 > `react-native/Libraries/Image/Image.android.js`

可以看到，尽管RN在0.18后全面转向了ES6，但组件Image仍然采用了ES5风格的JavaScript。

###属性的定义
Image在`propTypes`中定义了该组件支持的各种属性及其属性值的类型，这里我们着重介绍`source`属性和`style`属性的定义，而对于组件接收到的属性值的处理则放在了`render`函数中
####图片URI
组件Image自身定义了如下几种属性：
> - `source {uri: string}, number` : 表示图片资源的位置。
```javascript
source: PropTypes.oneOfType([//可以接受如下三种形式的资源位置
      PropTypes.shape({
      //`uri`是一个表示图片的资源标识的字符串，它可以是一个http地址或是一个本地文件路径
        uri: PropTypes.string,
      }),
      // 也可以是一个通过函数`require('./path/to/image.png')`获取到静态资源
      PropTypes.number,
      //也可以接受一个包含了多个图片uri的数组，在数组里可以指定每个图片显示的宽高
      PropTypes.arrayOf(
        PropTypes.shape({
          uri: PropTypes.string,
          width: PropTypes.number,
          height: PropTypes.number,
        }))
    ]),
```
> - `loadingIndicatorSource` : 表示在真正图片在加载过程中所显示的图片，在加载网络图片的场景下特别有用。
```javascript
loadingIndicatorSource: PropTypes.oneOfType([//该属性与source的定义相似，但是不支持多图。
      PropTypes.shape({
        uri: PropTypes.string,
      }),
      // Opaque type returned by require('./image.jpg')
      PropTypes.number,
    ]),
```

----
####组件样式`style`的定义
> - `style`:  定义了Image这个组件可以接收的样式。
```javascript
style: StyleSheetPropType(ImageStylePropTypes)
```
在`ImageStylePropTypes`中定义了开发者可以为Image设置用到的style属性：
```javascript
var ImageStylePropTypes = {
//引入其他公用属性
  ...LayoutPropTypes,
  ...ShadowPropTypesIOS,
  ...TransformPropTypes,
  //可以设置图片的调整模式
  resizeMode: ReactPropTypes.oneOf(Object.keys(ImageResizeMode)),
  backfaceVisibility: ReactPropTypes.oneOf(['visible', 'hidden']),
  //背景色
  backgroundColor: ColorPropType,
  //边框色
  borderColor: ColorPropType,
  //边框宽度
  borderWidth: ReactPropTypes.number,
  //边框圆角度数
  borderRadius: ReactPropTypes.number,
  overflow: ReactPropTypes.oneOf(['visible', 'hidden']),
  tintColor: ColorPropType,
  opacity: ReactPropTypes.number,

  overlayColor: ReactPropTypes.string,

  borderTopLeftRadius: ReactPropTypes.number,
  borderTopRightRadius: ReactPropTypes.number,
  borderBottomLeftRadius: ReactPropTypes.number,
  borderBottomRightRadius: ReactPropTypes.number,
};
```
ReactNative把`LayoutPropTypes`等一些公用的style属性提取出来，为了便于其他组件复用。

-----
####其他属性
> - `progressiveRenderingEnabled`：表示是否采用渐进式加载，渐进式加载时图片会从模糊到清晰渐渐呈现。

> - `fadeDuration`: 图片淡入淡出时间，毫秒。

> - `resizeMode`: 决定当组件尺寸和图片尺寸不成比例的时候如何调整图片的大小，目前支持以下几种模式:
 -  `cover`: 在保持图片宽高比的前提下缩放图片，直到宽度和高度都大于等于容器视图的尺寸（如果容器有-padding内衬的话，则相应减去）。`note`：这样图片完全覆盖甚至超出容器，容器中不留任何空白。
 -   `contain`: 在保持图片宽高比的前提下缩放图片，直到宽度和高度都小于等于容器视图的尺寸（如果容器有padding内衬的话，则相应减去）。`note`：这样图片完全被包裹在容器中，容器中可能留有空白
 -  `stretch`: 拉伸图片且不维持宽高比，直到宽高都刚好填满容器。
 - `center`: 居中不拉伸。

另外，Image还可以通过属性指定图片加载阶段的回调函数：
> - `onLoad`: 图片加载成功完成时调用此回调函数。
> - `onLoadStart` :图片加载开始时调用。
> - `onLoadEnd`：图片加载结束后，无论成功还是失败，调用此回调函数。

-------
###Mixin
在 React component 构建过程中，为了将同样的功能添加到多个组件当中，可以将这些通用的功能包装成一个mixin，然后导入到组件中。（ES6不支持mixin）
Image组件同样引入了Mixin对象：
```javascript
mixins: [NativeMethodsMixin]
```
Image所用到的主要功能是`setNativeProps`函数：
```javascript
setNativeProps: function (nativeProps) {
  if (process.env.NODE_ENV !== 'production') {
    warnForStyleProps(nativeProps, this.viewConfig.validAttributes);
  }
  var updatePayload = ReactNativeAttributePayload.create(nativeProps, this.viewConfig.validAttributes);

  UIManager.updateView(findNodeHandle(this), this.viewConfig.uiViewClassName, updatePayload);
}
```
我们都知道，React Diff会计算出Virtual Dom中真正变化的部分并进行渲染，而`setNativeProps`函数的作用在于直接将属性对象传递给Native层，不经过diff这个过程。这就意味着，如果不在接下来的render过程中包含这些属性，这些属性仍然会起作用。
那么，Image组件会传递哪些属性给Native呢？默认情况下，Image设置并传递了`viewConfig`对象：
```javascript
viewConfig: {
  uiViewClassName: 'RCTView',
  validAttributes: ReactNativeViewAttributes.RCTView,
}
```
而在组件的生命周期回调函数`componentWillMount`和`componentWillReceiveProps`则会更新属性值：
```javascript
_updateViewConfig: function(props) {
  if (props.children) {//有子组件
    this.viewConfig = {
      uiViewClassName: 'RCTView',
      validAttributes: ReactNativeViewAttributes.RCTView,
    };
  } else {
    this.viewConfig = {//无子组件
      uiViewClassName: 'RCTImageView',
      validAttributes: ImageViewAttributes,
    };
  }
},
componentWillMount: function() {
  this._updateViewConfig(this.props);
},

componentWillReceiveProps: function(nextProps) {
  this._updateViewConfig(nextProps);
},
```
可以看到，如果Image有子组件的情况下则会传递`ReactNativeViewAttributes.RCTView`,否则会传递`ImageViewAttributes`：
```javascript
var ImageViewAttributes = merge(ReactNativeViewAttributes.UIView, {
  src: true,
  loadingIndicatorSrc: true,
  resizeMode: true,
  progressiveRenderingEnabled: true,
  fadeDuration: true,
  shouldNotifyLoadEvents: true,
});
```


-----

###渲染
作为一个组件，Image首先需要对各种属性进行相应的处理工作，然后根据开发者设置的属性值，相应的业务场景以及组件状态渲染出相应的界面。这部分逻辑在`render`函数中定义。
####`source`属性解析
Image首先会对开发者传入的`source`进行解析，判断需要从哪里加载图片：
```javascript
const source = resolveAssetSource(this.props.source);
const loadingIndicatorSource = resolveAssetSource(this.props.loadingIndicatorSource);
```
来看一下`resolveAssetSource.js`中的路径解析逻辑：
```javascript
//由属性定义可知，这里传入的source值可以是一个uri对象，或者require函数返回的数字
function resolveAssetSource(source: any): ?ResolvedAssetSource {
  if (typeof source === 'object') {//如果是uri对象则不作处理直接返回
    return source;
  }
  //获取静态图片的基本信息
  var asset = AssetRegistry.getAssetByID(source);
  if (!asset) {
    return null;
  }

  const resolver = new AssetSourceResolver(getDevServerURL(), getBundleSourcePath(), asset);
  if (_customSourceTransformer) {
    return _customSourceTransformer(resolver);
  }
  return resolver.defaultAsset();
}
```
这里创建了`AssetSourceResolver`，并传入了三个参数：
- `getDevServerURL`: 如果从Node服务器加载JS，则返回相应路径；
```javascript
function getDevServerURL(): ?string {
  if (_serverURL === undefined) {
    var scriptURL = SourceCode.scriptURL;//调用Native module获取JSbundle路径
    var match = scriptURL && scriptURL.match(/^https?:\/\/.*?\//);
    if (match) {
      // 从网络中获取JS
      _serverURL = match[0];
    } else {
      // 从本地加载JS文件
      _serverURL = null;
    }
  }
  return _serverURL;
}
```

- `getBundleSourcePath`：如果用户自定义了JS文件路径，则返回无协议头的路径；否则返回空
```javascript
function getBundleSourcePath(): ?string {
  if (_bundleSourcePath === undefined) {
    const scriptURL = SourceCode.scriptURL;
    if (!scriptURL) {//未传递JS路径
      _bundleSourcePath = null;
      return _bundleSourcePath;
    }
    if (scriptURL.startsWith('assets://')) {
      // 未指定JS文件路径，离线时默认从asset目录加载
      _bundleSourcePath = null;
      return _bundleSourcePath;
    }
    if (scriptURL.startsWith('file://')) {
      //如果开发者指定了JS的目录，则返回去除协议头部的文件路径
      _bundleSourcePath = scriptURL.substring(7, scriptURL.lastIndexOf('/') + 1);
    } else {
      _bundleSourcePath = scriptURL.substring(0, scriptURL.lastIndexOf('/') + 1);
    }
  }

  return _bundleSourcePath;
}
```

- `asset`：使用`AssetRegistry.getAssetByID`返回的图片基本信息

这里需要注意的是，RN允许开发者在Native端自定义JS的加载路径，在JS端可以调用`SourceCode.scriptURL`来获取。如果开发者未指定JSbundle的路径，则在离线环境下返回asset目录，开发环境下从node服务器读取。

通过上述三个参数：JSbundle在node服务器的路径，JS在本地的路径以及图片的基本信息，RN构建了`AssetSourceResolver`对象,并调用了`defaultAsset()`：
```javascript
defaultAsset(): ResolvedAssetSource {
    if (this.isLoadedFromServer()) {//从服务器加载，返回url
      return this.assetServerURL();
    }
    if (Platform.OS === 'android') {
      return this.isLoadedFromFileSystem() ?
        this.drawableFolderInBundle() ://从自定义路径加载
        this.resourceIdentifierWithoutScale();//从asset目录加载
    } else {
      return this.scaledAssetPathInBundle();
    }
  }
```
`defaultAsset()`方法其实根据具体的场景返回了一个包含了图片加载所需信息的对象而已。
值得我们注意的是，如果开发者未指定JSbundle的加载路径，则会调用`resourceIdentifierWithoutScale`方法，该方法中会调用`assetPathUtils`对图片的加载路径进行处理，把`图片目录/图片名称`的格式处理成`图片目录_图片名称`的形式。（这是由于打包命令同样会调用`assetPathUtils`对资源文件进行处理）


根据上述代码的分析，对于source属性的处理解析而言，我们可以得出结论如下：
- 通过uri指定图片路径，则对source属性不做处理，直接返回包含了uri字符串的对象。
- 通过require()请求静态资源的方式加载图片，则会根据加载位置的不同返回路径不同的对象：
 - 开发环境下，从node server加载JS ，则返回类似`http://localhost:8081/index.android.bundle?platform=android&dev=true`的路径。
 - 离线状态下，如果未指定JS路径，则返回经过处理的asset路径，例如：
 - 如果用户未指定JS路径，则返回用户自定义的路径，例如`file:///sdcard/AwesomeModule/drawable-mdpi/icon.png`.

-----
####`style`处理
通过`resolveAssetSource`对source属性解析后，我们得到了一个包含了图片加载信息的对象，并利用该对象中的信息创建了要渲染的样式`style`，并将source属性封装为一个含有uri对象的数组：
```javascript
const {width, height} = source;
//读取到图片的宽高，Image定义的style，开发者自定义的style
style = flattenStyle([{width, height}, styles.base, this.props.style]);
sources = [{uri: source.uri}];
```
函数`flattenStyle`接收了一个包含各类样式对象的数组。通过递归，`flattenStyle`把数组中的样式都进行了整理，合并为一个统一的`style`对象。

----
####渲染组件
在组件渲染之前，Image把之前处理过的style，source以及开发者传入的组件的属性进行了整理合并：
```javascript
const nativeProps = merge(this.props, {
        style,//把style解析合并传递给native
        shouldNotifyLoadEvents: !!(onLoadStart || onLoad || onLoadEnd),
        src: sources,//历史原因导致Native端对应的属性名为src，但开发者在react端要用source
        loadingIndicatorSrc: loadingIndicatorSource ? loadingIndicatorSource.uri : null,
      });
```

通过对开发者传入的组件的属性，样式以及source等属性进行合并后，根据不同的业务场景渲染出相应的组件：
```javascript
if (nativeProps.children) {
    //如果Image组件中包含了其他组件
    const containerStyle = filterObject(style, (val, key) => !ImageSpecificStyleKeys.has(key));
    const imageStyle = filterObject(style, (val, key) => ImageSpecificStyleKeys.has(key));
    const imageProps = merge(nativeProps, {
      style: [imageStyle, styles.absoluteImage],
      children: undefined,
        });

    return (
      <View style={containerStyle}>
        <RKImage {...imageProps}/>
        {this.props.children}
      </View>
    );
  } else {
    if (this.context.isInAParentText) {//如果是TextView内嵌Image
      return <RCTTextInlineImage {...nativeProps}/>;
    } else {//渲染普通情况下的Image组件
      return <RKImage {...nativeProps}/>;
    }
  }
```
Image组件提供了两种图片展示方式，内嵌于TextView中的图片和普通图片：
```javascript
var RKImage = requireNativeComponent('RCTImageView', Image, cfg);
var RCTTextInlineImage = requireNativeComponent('RCTTextInlineImage', Image, cfg);
```
可以看到，内嵌图片的情况下引用了Native端实现的`RCTTextInlineImage`组件，而普通图片则使用了`RCTImageView`。
通过对Image组件在JavaScript端代码进行分析，我们了解该组件是如何定义属性以及如何根据开发者所设置的属性渲染出不同的效果。而对于这些属性所设置的效果进行显示和响应则要靠Native层的实现。

------
##Native端
我们知道，ReactNative的图片相关处理工作是交给图片库Fresco完成的，这里我们不去深究细节，只去关注组件本身的实现逻辑。Image组件在native端实的现位于：
> `.../react/views/image/ReactImageManager.java`
对于自定义的组件，在Native端需要创建一个`ReactImageManager`负责进行Native组件的创建，相关属性的管理，事件的响应等工作，而真正的具体实现操作则要放在Native组件中完成。
同样，Image组件通过`ReactImageManager`来完成Native组件`ReactImageView`的创建和管理：
```java
public ReactImageView createViewInstance(ThemedReactContext context) {
  return new ReactImageView(
      context,
      getDraweeControllerBuilder(),
      getCallerContext());
}
//与react端关联
public String getName() {
    return "RCTImageView";
}
```
通过注解`@ReactProp`或`ReactPropGroup`，`ReactImageManager`导出了暴露给React端的属性设置方法，将相关属性值交给Native组件`ReactImageView`，以`src`属性为例：
```java
@ReactProp(name = "src")
public void setSource(ReactImageView view, @Nullable ReadableArray sources) {
  view.setSource(sources);
}
public void setSource(@Nullable ReadableArray sources) {
  mSources.clear();
  if (sources != null && sources.size() != 0) {
    if (sources.size() == 1) {
      mSources.add(new ImageSource(getContext(), sources.getMap(0).getString("uri")));
    } else {
      for (int idx = 0; idx < sources.size(); idx++) {
        ReadableMap source = sources.getMap(idx);
        mSources.add(new ImageSource(
          getContext(),
          source.getString("uri"),
          source.getDouble("width"),
          source.getDouble("height")));
      }
    }
  }
  mIsDirty = true;
}
```
`ReactImageView`维护了相关的属性对象，并在接收到React端传递过来的属性值后进行相应处理解析，但此时不会刷新组件，只会标记一下该组件需要刷新。而统一的刷新操作则放在`ReactImageManager`的`onAfterUpdateTransaction`方法中进行：
```java
protected void onAfterUpdateTransaction(ReactImageView view) {
  super.onAfterUpdateTransaction(view);
  view.maybeUpdateView();
}
```
具体的刷新实现是通过图片库Fresco进行实现的，这里不再详细分析。

##总结
通过对整个Image组件的分析，我们可以看到React端定义了组件可以使用的属性类型和事件，并在渲染时将它们传递给Native层；Native层通过`ViewManager`来对Native组件进行对传递过来的属性值进行管理，而具体的实现则放在Native组件中。