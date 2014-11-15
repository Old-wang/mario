# "html" ��

### Ŀ��

`html` ���ù����� DOM Ԫ�ؿ�����ʾ��Ĳ�����ָ���� HTML��

ͨ�����������ͼģ���е�ֵʵ����������ϣ����Ⱦ�� HTML ��ǵ��ַ���ʱ�ǳ����á�

ʾ��

```html
<div data-bind="html: details"></div>
 
<script type="text/javascript">
    var viewModel = {
        details: ko.observable() // Initially blank
    };
    viewModel.details("<em>For further details, view the report <a href='report.html'>here</a>.</em>"); // HTML content appears
</script>
```

### ����

* ������

  KO �����ǰ�ĵ����ݲ�ʹ�� jQuery �� `html` �������� jQuery ������ʱͨ�����ַ�������Ϊ HTML �ڵ㲢��ÿ���ڵ���Ϊ
��Ԫ�ص��ӽڵ㸽������Ԫ�ص���������Ϊ��Ĳ���ֵ��

  ����ò�����һ�� observable ֵ���ð󶨻���ֵ���ʱ����Ԫ�ص����ݡ����������� observable��
����������Ԫ�ص�����һ������Ҳ������¡�

  ������ṩ�����ֻ��ַ�����Ϊ�Ķ���(�紫����һ�����������)���� `innerHTML` ���ȼ��� `yourParameter.toString()`��

* �������

   * û��
   
### ע������ HTML ת��

���ڸð�ʹ�� `innerHTML` ���������Ԫ�ص����ݣ���Ӧ��С�Ĳ��ڲ����ŵ�ģ��ֵ��ʹ������
��Ϊ�������ܿ����ű�ע�빥���Ŀ����ԡ�
����㲻�ܱ�֤������ʾ�İ�ȫ��(�������ڴ洢��������ݿ���һ����ͬ�û�������)��
�������ʹ�� [text ��](./text-binding.md)������ת��ʹ�� `innerText` �� `textContent` ������Ԫ�ص��ı�ֵ��

### ����

û�У����˺��� Knockout ��Ȿ��