# �� computed observables

*��* computed observables �� Knockout 3.2.0 �����룬Ϊ�����Ӧ���ṩ�˱ȳ��� computed observables ����
�����ܺ��ڴ����ơ�������Ϊһ�� *��* computed observable ������û�ж�����ʱ��ά�����������Ķ��ġ�
������ԣ�

* �����ٱ�Ӧ�������õ������������ڵ� computed observables **�������ڴ�й¶**
* ͨ�������¼���ֵû�б��۲�� computed observables ��ֵ�� **���ټ��㿪֧**

һ�� *��* computed observables ���Զ��������Ƿ���ڶ�����������״̬���л���

1. ��û�ж�����ʱ�������� **sleeping**�������� *sleeping* ״̬�������������ж��������Ķ��ġ�
�����״̬������������ֵ�����ж����κη��ʵ��� observables 
(������ȷʵ���м��������� `getDependenciesCount()` ������ȷ��)��
��� computed observable ��ֵ���� *sleeping* ʱ����ȡ�������ǻ�������ֵȷ����ֵ�����µġ�
2. �����ڶ�����ʱ�������� **listening**�������� *listening* ״̬��������������ֵ�����������κη��ʵ��� observables��
���״̬�£����Ĳ����ͺͳ���� computed observable һ������ [���������������](./computed-dependency-tracking.md) ��������

## Why ��pure��?

���Ǵ� [������](http://en.wikipedia.org/wiki/Pure_function) �ĸ����н���������
��Ϊ�������ͨ������������ֵ��������һ���� *������* �� computed observables ��

1. �Ը� computed observable ����ֵ���ᵼ���κθ�����
2. �� computed observable ��ֵ���������ֵ�Ĵ���������"����"����Ϣ����ֵӦ�ý�����Ӧ�������� observables ��ֵ��
��Ӧ�ڴ������Ķ��壬���������������

## �﷨

����һ�� *��* computed observable �ı�׼������ʹ�� `ko.pureComputed` ��

```javascript
this.fullName = ko.pureComputed(function() {
    return this.firstName() + " " + this.lastName();
}, this);
```

���ߣ������ʹ�� `ko.computed` �� `pure` ѡ�

```javascript
this.fullName = ko.computed(function() {
    return this.firstName() + " " + this.lastName();
}, this, { pure: true });
```

�����﷨����鿴 [computed observable �ο�](./computed-reference.md)��

## ��ʱʹ�� *��* computed observable

����Խ� *��* �������õ��κ���ѭ [������ָ��](./computed-pure.md) �� computed observable �ϡ�
Ȼ������Ӧ�õ���Ƴ־û���ͼģ�ͱ���ʱ����ͼ����ͼģ����ʹ�ú͹����Ӧ�������ʱ�����á�
�ڳ־û���ͼģ����ʹ�� *��* computed observable �ṩ�˼�����������(������������)��
����ʱ��ͼģ����ʹ�������ṩ���ڴ�������ơ�

��������򵼽ӿ�ʾ���У� `fullName` �� computed �������һ�����󶨵���ͼ�������������һ������ʱ�����¡�

Դ�룺��ͼ

```html
<div class="log" data-bind="text: computedLog"></div>
<!--ko if: step() == 0-->
    <p>First name: <input data-bind="textInput: firstName" /></p>
<!--/ko-->
<!--ko if: step() == 1-->
    <p>Last name: <input data-bind="textInput: lastName" /></p>
<!--/ko-->
<!--ko if: step() == 2-->
    <div>Prefix: <select data-bind="value: prefix, options: ['Mr.', 'Ms.','Mrs.','Dr.']"></select></div>
    <h2>Hello, <span data-bind="text: fullName"> </span>!</h2>
<!--/ko-->
<p><button type="button" data-bind="click: next">Next</button></p>
```

Դ�룺 ��ͼģ��

```javascript
function AppData() {
    this.firstName = ko.observable('John');
    this.lastName = ko.observable('Burns');
    this.prefix = ko.observable('Dr.');
    this.computedLog = ko.observable('Log: ');
    this.fullName = ko.pureComputed(function () {
        var value = this.prefix() + " " + this.firstName() + " " + this.lastName();
        // Normally, you should avoid writing to observables within a pure computed 
        // observable (avoiding side effects). But this example is meant to demonstrate 
        // its internal workings, and writing a log is a good way to do so.
        this.computedLog(this.computedLog.peek() + value + '; ');
        return value;
    }, this);
 
    this.step = ko.observable(0);
    this.next = function () {
        this.step(this.step() === 2 ? 0 : this.step()+1);
    };
};
ko.applyBindings(new AppData());
```

## ��ʱ��ʹ�� *��* computed observable

### ������

�㲻Ӧ������Ϊ�����������ʱִ��ĳ�������� computed observable ��ʹ�� *��* ���ԡ��磺

* ��ʹ�� computed observable �����ڶ�� observables ���лص�

```javascript
ko.computed(function () {
    var cleanData = ko.toJS(this);
    myDataClient.update(cleanData);
}, this);
```

* �ڰ󶨵� `init` �����У�ʹ�� computed observable ����������Ԫ��

```javascript
ko.computed({
    read: function () {
        element.title = ko.unwrap(valueAccessor());
    },
    disposeWhenNodeIsRemoved: element
});
```

�㲻������ֵ��������Ҫ�ĸ�����ʱʹ�� pure computed ��ԭ�򣬼���������ֻҪ�� computed 
û�л�Ķ�����(��������� sleeping) ��ֵ�����������С�
������������ʱ��ֵ���������к���Ҫ����Ҫʹ�� [����� observable](./computedObservable.md)��

### ����

ĳЩ����¸� computed observable ʹ�� pure ���Է����ᵼ�¸��ߵļ��㿪֧��
����� computed observable ������ĳ�������б��ʱ������ֵ��
�� *��* computed observable �� sleeping ״̬��ÿ�α����ʵ�ʱ�ͽ��� listening ״̬ʱ����������ֵ��
��˾ʹ���ʹ�� pure ���Ե� computed observable �ȳ��������ֵ��Ƶ���Ŀ����ԡ�