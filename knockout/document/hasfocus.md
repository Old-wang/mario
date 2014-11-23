# "hasFocus" ��

### Ŀ��

`hasFocus` �󶨽�һ�� DOM Ԫ�ص� focus ״̬������ͼģ�͵������ϡ�
����һ��˫��󶨣���ˣ�

* ����㽫��ͼģ����������Ϊ `true` �� `false`��������Ԫ�ؽ��õ������ʧȥ����
* ����û��ֶ�������Ԫ�صõ���ʧȥ���㣬��ͼģ�����Խ���Ӧ�ر�����Ϊ `true` �� `false`

�����㹹�����ӵı������пɱ༭Ԫ�ض�̬�س��֣�����ϣ�������û�Ӧ�������￪ʼд��
����Ӧ�س���λ�ã�ʱ�ǳ����á�

### ʾ��1������ʾ��

���ʾ���򵥵����ı���ǰ�õ�����ʱ��ʾһ����Ϣ���������ʹ�ð�ť�����ʽ�ش������㡣

Դ�룺 ��ͼ

```html
<input data-bind="hasFocus: isSelected" />
<button data-bind="click: setIsSelected">Focus programmatically</button>
<span data-bind="visible: isSelected">The textbox has focus</span>
```

```javascript
var viewModel = {
    isSelected: ko.observable(false),
    setIsSelected: function() { this.isSelected(true) }
};
ko.applyBindings(viewModel);
```

### ʾ��2�� �������༭

��Ϊ `hasFocus` �󶨿�˫������(���ù�����ֵ��ʹԪ�صõ���ʧȥ���㣻��Ԫ���ϵõ���ʧȥ����Ҳ�����ù�����ֵ)��
�����Գ�Ϊ���� "�༭" ģʽ�ķ���ط�ʽ��
��� UI ����ģ�͵� `editing` ����ֵ����ʾһ�� `<span>` ��һ�� `<input>` Ԫ�ء�
`<input>` Ԫ��ʧȥ����Ὣ `editing` ����Ϊ `false`������ UI �ֻ��뿪 "�༭" ģʽ��

Դ�룺��ͼ

```html
<p>
    Name: 
    <b data-bind="visible: !editing(), text: name, click: edit">&nbsp;</b>
    <input data-bind="visible: editing, value: name, hasFocus: editing" />
</p>
<p><em>Click the name to edit it; click elsewhere to apply changes.</em></p>
```

Դ�룺��ͼģ��

```javascript
function PersonViewModel(name) {
    // Data
    this.name = ko.observable(name);
    this.editing = ko.observable(false);
         
    // Behaviors
    this.edit = function() { this.editing(true) }
}
 
ko.applyBindings(new PersonViewModel("Bert Bertington"));
```

### ����

  * ������
  
  ���� `true`(�������߼���ֵ)��ʹ����Ԫ�صõ����㡣���򣬹���Ԫ�ؽ�ʧȥ���㡣
  
  ���û��ֶ���Ԫ�صõ���ʧȥ���㣬���ֵҲ�ᱻ��Ӧ������Ϊ `true` �� `false`��
  
  ������ṩ��ֵ�� observable�� `hasFocus` �󶨽����� observable ֵ���ʱ����Ԫ�ؽ���״̬��

  * �������
  
    * û��

### ����

û�У����˺��� Knockout ��Ȿ��