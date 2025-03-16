# एस्पेरांतो पाठ प्रसंस्करण अनुप्रयोग: तकनीकी विवरण

## विषय-सूची

1. [वास्तुकला अवलोकन](#वास्तुकला-अवलोकन)
   - [फाइल संरचना](#फाइल-संरचना)
   - [प्रमुख घटक और उनके कार्य](#प्रमुख-घटक-और-उनके-कार्य)
   - [डेटा प्रवाह](#डेटा-प्रवाह)

2. [मूल डेटा संरचनाएँ](#मूल-डेटा-संरचनाएँ)
   - [वर्ण मैपिंग शब्दकोश](#वर्ण-मैपिंग-शब्दकोश)
   - [प्रतिस्थापन सूचियों की संरचना](#प्रतिस्थापन-सूचियों-की-संरचना)
   - [प्लेसहोल्डर तंत्र](#प्लेसहोल्डर-तंत्र)

3. [मुख्य प्रसंस्करण प्रवाह](#मुख्य-प्रसंस्करण-प्रवाह)
   - [उपयोगकर्ता इनपुट हैंडलिंग](#उपयोगकर्ता-इनपुट-हैंडलिंग)
   - [पाठ पूर्व-प्रसंस्करण](#पाठ-पूर्व-प्रसंस्करण)
   - [प्रतिस्थापन अनुप्रयोग](#प्रतिस्थापन-अनुप्रयोग)
   - [पश्च-प्रसंस्करण](#पश्च-प्रसंस्करण)
   - [आउटपुट जनरेशन](#आउटपुट-जनरेशन)

4. [समानांतर प्रसंस्करण कार्यान्वयन](#समानांतर-प्रसंस्करण-कार्यान्वयन)
   - [पाठ विभाजन](#पाठ-विभाजन)
   - [प्रक्रियाओं का प्रबंधन](#प्रक्रियाओं-का-प्रबंधन)
   - [परिणाम एकत्रीकरण](#परिणाम-एकत्रीकरण)

5. [प्रतिस्थापन तर्क](#प्रतिस्थापन-तर्क)
   - [सुरक्षित प्रतिस्थापन एल्गोरिथ्म](#सुरक्षित-प्रतिस्थापन-एल्गोरिथ्म)
   - [वर्ण रूपांतरण](#वर्ण-रूपांतरण)
   - [प्लेसहोल्डर उपयोग](#प्लेसहोल्डर-उपयोग)
   - [% और @ के साथ विशेष प्रतिस्थापन](#-और--के-साथ-विशेष-प्रतिस्थापन)

6. [JSON जनरेशन](#json-जनरेशन)
   - [CSV पार्सिंग और प्रसंस्करण](#csv-पार्सिंग-और-प्रसंस्करण)
   - [मूल शब्द निष्कर्षण](#मूल-शब्द-निष्कर्षण)
   - [प्रतिस्थापन नियम निर्माण](#प्रतिस्थापन-नियम-निर्माण)
   - [JSON संरचना](#json-संरचना)

7. [उन्नत विशेषताएँ](#उन्नत-विशेषताएँ)
   - [कस्टम आउटपुट फॉर्मेट](#कस्टम-आउटपुट-फॉर्मेट)
   - [रूबी एनोटेशन](#रूबी-एनोटेशन)
   - [वर्ण चौड़ाई समायोजन](#वर्ण-चौड़ाई-समायोजन)

8. [प्रदर्शन अनुकूलन](#प्रदर्शन-अनुकूलन)
   - [कैशिंग तकनीक](#कैशिंग-तकनीक)
   - [समानांतर प्रसंस्करण के प्रभाव](#समानांतर-प्रसंस्करण-के-प्रभाव)
   - [मेमोरी प्रबंधन](#मेमोरी-प्रबंधन)

## वास्तुकला अवलोकन

एस्पेरांतो पाठ प्रसंस्करण अनुप्रयोग एक स्ट्रीमलिट-आधारित वेब एप्लिकेशन है जो एस्पेरांतो पाठ को प्रसंस्करण करके उसे विभिन्न प्रारूपों में रूपांतरित करता है। अनुप्रयोग की वास्तुकला मॉड्यूलर है, विभिन्न कार्यक्षमताओं को अलग-अलग फाइलों में विभाजित किया गया है।

### फाइल संरचना

अनुप्रयोग चार प्रमुख फाइलों से बना है:

1. **main.py**: मुख्य स्ट्रीमलिट एप्लिकेशन फाइल जो उपयोगकर्ता इंटरफेस और मुख्य प्रसंस्करण तर्क प्रदान करती है।

2. **एस्पेरान्तो पाठ को स्ट्रिंग (कांजी) से प्रतिस्थापित करने हेतु JSON फ़ाइल बनाने का पृष्ठ.py**: स्ट्रीमलिट "pages" फ़ोल्डर में एक फाइल जो प्रतिस्थापन JSON फाइल बनाने के लिए एक अलग पृष्ठ प्रदान करती है।

3. **esp_text_replacement_module.py**: एक उपयोगिता मॉड्यूल जो एस्पेरांतो पाठ प्रतिस्थापन के लिए फ़ंक्शन प्रदान करता है।

4. **esp_replacement_json_make_module.py**: एक और उपयोगिता मॉड्यूल जो प्रतिस्थापन JSON फाइल बनाने के लिए विशेष फ़ंक्शन प्रदान करता है।

### प्रमुख घटक और उनके कार्य

अनुप्रयोग के प्रमुख घटक और उनके कार्य इस प्रकार हैं:

**1. मुख्य एप्लिकेशन (main.py)**
- उपयोगकर्ता इंटरफेस का प्रबंधन करता है
- उपयोगकर्ता इनपुट प्राप्त करता है
- प्रतिस्थापन JSON फाइल लोड करता है
- प्रसंस्करण कार्य का समन्वय करता है
- परिणाम प्रदर्शित करता है

**2. JSON जनरेटर पेज (एस्पेरान्तो पाठ को स्ट्रिंग...)**
- CSV फाइलों से प्रतिस्थापन नियम इम्पोर्ट करता है
- प्रतिस्थापन नियमों को प्रसंस्करित करता है
- संयुक्त JSON फाइल बनाता है

**3. पाठ प्रतिस्थापन मॉड्यूल (esp_text_replacement_module.py)**
- वर्ण रूपांतरण फ़ंक्शन प्रदान करता है
- प्रतिस्थापन एल्गोरिथ्म कार्यान्वित करता है
- प्लेसहोल्डर प्रबंधन करता है
- समानांतर प्रसंस्करण तंत्र प्रदान करता है

**4. JSON जनरेशन उपयोगिता (esp_replacement_json_make_module.py)**
- CSV से डेटा संसाधित करता है
- रूबी एनोटेशन का उपयोग करके आउटपुट फॉर्मेटिंग करता है
- वर्ण चौड़ाई मापन और समायोजन प्रदान करता है
- JSON जनरेशन को सहायता प्रदान करता है

### डेटा प्रवाह

अनुप्रयोग में डेटा प्रवाह का अवलोकन:

1. उपयोगकर्ता एक एस्पेरांतो पाठ और प्रतिस्थापन JSON फाइल (अपलोड या डिफॉल्ट) प्रदान करता है
2. मुख्य एप्लिकेशन JSON फाइल से प्रतिस्थापन नियम लोड करता है
3. पाठ को प्रसंस्करण के लिए तैयार किया जाता है (एस्पेरांतो वर्ण रूपांतरण, स्पेस नॉर्मलाइजेशन)
4. प्रतिस्थापन पहले प्लेसहोल्डर और फिर वास्तविक प्रतिस्थापनों के साथ लागू किया जाता है
5. परिणाम स्वरूपित और प्रदर्शित किया जाता है, डाउनलोड के लिए उपलब्ध होता है

JSON जनरेटर पेज प्रवाह:
1. उपयोगकर्ता CSV फाइल और वैकल्पिक रूप से विखंडन/प्रतिस्थापन नियम प्रदान करता है
2. जनरेटर तीन प्रकार के प्रतिस्थापन नियम बनाता है (वैश्विक, स्थानीय, और 2-वर्ण)
3. नियम एक संयुक्त JSON में बचाए जाते हैं

## मूल डेटा संरचनाएँ

### वर्ण मैपिंग शब्दकोश

अनुप्रयोग एस्पेरांतो वर्णों के विभिन्न प्रतिनिधित्वों को परिवर्तित करने के लिए वर्ण मैपिंग शब्दकोश का उपयोग करता है। ये पायथन डिक्शनरीज हैं जो एस्पेरांतो के विशेष वर्णों के बीच मैपिंग प्रदान करते हैं:

```python
# x प्रणाली से circumflex प्रणाली (जैसे cx → ĉ)
x_to_circumflex = {
    'cx': 'ĉ', 'gx': 'ĝ', 'hx': 'ĥ', 'jx': 'ĵ', 'sx': 'ŝ', 'ux': 'ŭ',
    'Cx': 'Ĉ', 'Gx': 'Ĝ', 'Hx': 'Ĥ', 'Jx': 'Ĵ', 'Sx': 'Ŝ', 'Ux': 'Ŭ'
}

# circumflex प्रणाली से x प्रणाली (ĉ → cx)
circumflex_to_x = {
    'ĉ': 'cx', 'ĝ': 'gx', 'ĥ': 'hx', 'ĵ': 'jx', 'ŝ': 'sx', 'ŭ': 'ux',
    'Ĉ': 'Cx', 'Ĝ': 'Gx', 'Ĥ': 'Hx', 'Ĵ': 'Jx', 'Ŝ': 'Sx', 'Ŭ': 'Ux'
}

# ऐसे ही अन्य वर्ण मैपिंग शब्दकोश हैं (x_to_hat, hat_to_x, आदि)
```

इसके अलावा, अनुप्रयोग Unicode वर्ण चौड़ाई को मापने के लिए एक शब्दकोश का उपयोग करता है:

```python
# Unicode वर्ण चौड़ाई शब्दकोश (JSON से लोड)
with open("./Appの运行に使用する各类文件/Unicode_BMP全范围文字幅(宽)_Arial16.json", "r", encoding="utf-8") as fp:
    char_widths_dict = json.load(fp)
```

### प्रतिस्थापन सूचियों की संरचना

प्रतिस्थापन नियम तीन सूचियों में संगठित किए गए हैं:

1. **replacements_final_list**: वैश्विक प्रतिस्थापन के लिए
2. **replacements_list_for_localized_string**: स्थानीय (@...@) प्रतिस्थापन के लिए
3. **replacements_list_for_2char**: दो-वर्ण मूल शब्दों के लिए विशेष प्रतिस्थापन

प्रत्येक सूची में (old_text, new_text, placeholder) ट्रिपल्स शामिल हैं।

इसके अलावा, अनुप्रयोग में दो-वर्ण मूल शब्द प्रतिस्थापन के लिए विशेष सूचियां हैं:

```python
# प्रत्यय 2-वर्ण मूल
suffix_2char_roots = ['ad', 'ag', 'am', 'ar', 'as', ...]

# उपसर्ग 2-वर्ण मूल
prefix_2char_roots = ['al', 'am', 'av', 'bo', 'di', ...]

# स्वतंत्र 2-वर्ण मूल
standalone_2char_roots = ['al', 'ci', 'da', 'de', 'di', ...]
```

### प्लेसहोल्डर तंत्र

प्लेसहोल्डर तंत्र अनुप्रयोग की प्रतिस्थापन रणनीति का एक महत्वपूर्ण हिस्सा है। यह विभिन्न प्रकार के प्लेसहोल्डर्स का उपयोग करता है:

1. **स्किपिंग प्रतिस्थापन के लिए प्लेसहोल्डर**: %...% से घिरे पाठ को अस्थायी रूप से बचाने के लिए
2. **स्थानीयकृत प्रतिस्थापन के लिए प्लेसहोल्डर**: @...@ से घिरे पाठ के लिए
3. **वैश्विक प्रतिस्थापन के लिए प्लेसहोल्डर**: पाठ में प्रतिस्थापनों को चरणों में लागू करने के लिए
4. **2-वर्ण मूल शब्द प्रतिस्थापन के लिए प्लेसहोल्डर**: 2-वर्ण मूल शब्दों के प्रतिस्थापन के लिए

प्लेसहोल्डर्स फाइलों से लोड किए जाते हैं:

```python
placeholders_for_skipping_replacements = import_placeholders(
    './Appの运行に使用する各类文件/占位符(placeholders)_%1854%-%4934%_文字列替换skip用.txt'
)
placeholders_for_localized_replacement = import_placeholders(
    './Appの运行に使用する各类文件/占位符(placeholders)_@5134@-@9728@_局部文字列替换结果捕捉用.txt'
)
```

## मुख्य प्रसंस्करण प्रवाह

### उपयोगकर्ता इनपुट हैंडलिंग

मुख्य एप्लिकेशन (main.py) में, प्रसंस्करण का पहला चरण उपयोगकर्ता इनपुट को प्राप्त करना और JSON फाइल को लोड करना है:

```python
# उपयोगकर्ता विकल्प से JSON लोड करें
if selected_option == "デフォルトを使用する":
    default_json_path = "./Appの运行に使用する各类文件/最终的な替换用リスト(列表)(合并3个JSON文件).json"
    (replacements_final_list,
     replacements_list_for_localized_string,
     replacements_list_for_2char) = load_replacements_lists(default_json_path)
else:
    # उपयोगकर्ता द्वारा अपलोड की गई JSON लोड करें
    # ...

# इनपुट स्रोत विकल्प
source_option = st.radio(
    "आप इनपुट पाठ कैसे देना चाहेंगे?",
    source_options,
    format_func=lambda x: "मैन्युअल इनपुट" if x == "手動入力" else "फ़ाइल अपलोड"
)
```

उपयोगकर्ता फ्री-टेक्स्ट इनपुट या टेक्स्ट फाइल अपलोड प्रदान कर सकता है। इसके अलावा, वे आउटपुट फॉर्मेट और एस्पेरांतो वर्ण प्रतिनिधित्व भी चुन सकते हैं।

### पाठ पूर्व-प्रसंस्करण

पाठ को मुख्य प्रतिस्थापन से पहले उपयुक्त रूपांतर से गुजरना पड़ता है:

1. **स्पेस नॉर्मलाइजेशन**: सभी हाफ-विड्थ स्पेस के प्रकारों को आम ASCII स्पेस में बदलना
2. **वर्ण रूपांतरण**: एस्पेरांतो वर्णों को एक सुसंगत प्रारूप में बदलना

```python
# पूर्व-प्रसंस्करण के लिए फ़ंक्शन से अंश
def unify_halfwidth_spaces(text: str) -> str:
    """
    हाफ-विड्थ स्पेस वर्णों को मानक ASCII स्पेस में बदलना
    """
    pattern = r"[\u00A0\u2002\u2003\u2004\u2005\u2006\u2007\u2008\u2009\u200A]"
    return re.sub(pattern, " ", text)

def convert_to_circumflex(text: str) -> str:
    """
    एस्पेरांतो वर्णों को circumflex प्रारूप में परिवर्तित करना
    """
    text = replace_esperanto_chars(text, hat_to_circumflex)
    text = replace_esperanto_chars(text, x_to_circumflex)
    return text
```

### प्रतिस्थापन अनुप्रयोग

प्रतिस्थापन अनुप्रयोग मुख्य प्रसंस्करण चरण है और कई सब-स्टेप्स शामिल करता है:

1. **प्लेसहोल्डर के साथ सुरक्षित प्रतिस्थापन**:
   - % से घिरे क्षेत्रों को सुरक्षित रखना
   - @ से घिरे क्षेत्रों के लिए स्थानीय प्रतिस्थापन लागू करना
   - मुख्य वैश्विक प्रतिस्थापन
   - 2-वर्ण मूल शब्द प्रतिस्थापन

2. **वर्ण प्रारूप रूपांतरण**: उपयोगकर्ता की पसंद के आधार पर एस्पेरांतो विशेष वर्णों को स्वरूपित करना

```python
# प्रतिस्थापन प्रवाह के मुख्य भाग
if submit_btn:
    # पूर्व में चुने गए पाठ को सत्र स्टेट में संग्रहीत करें
    st.session_state["text0_value"] = text0

    # उपयोगकर्ता चयन के आधार पर समानांतर या क्रमिक प्रसंस्करण का उपयोग करें
    if use_parallel:
        processed_text = parallel_process(
            text=text0,
            num_processes=num_processes,
            placeholders_for_skipping_replacements=placeholders_for_skipping_replacements,
            replacements_list_for_localized_string=replacements_list_for_localized_string,
            placeholders_for_localized_replacement=placeholders_for_localized_replacement,
            replacements_final_list=replacements_final_list,
            replacements_list_for_2char=replacements_list_for_2char,
            format_type=format_type
        )
    else:
        processed_text = orchestrate_comprehensive_esperanto_text_replacement(
            # समान पैरामीटर...
        )

    # उपयोगकर्ता चयन के आधार पर वर्ण प्रारूप को लागू करें
    if letter_type == '上付き文字':
        processed_text = replace_esperanto_chars(processed_text, x_to_circumflex)
        processed_text = replace_esperanto_chars(processed_text, hat_to_circumflex)
    elif letter_type == '^形式':
        processed_text = replace_esperanto_chars(processed_text, x_to_hat)
        processed_text = replace_esperanto_chars(processed_text, circumflex_to_hat)
```

### पश्च-प्रसंस्करण

प्रतिस्थापन के बाद, पाठ को उपयोगकर्ता द्वारा चुने गए आउटपुट फॉर्मेट के अनुसार पोस्ट-प्रोसेस किया जाता है:

```python
# HTML हेडर और फुटर लागू करें
processed_text = apply_ruby_html_header_and_footer(processed_text, format_type)
```

इस चरण में HTML हेडर और CSS स्टाइल्स जोड़ना शामिल है, जो आउटपुट फॉर्मेट के अनुसार भिन्न हो सकते हैं।

### आउटपुट जनरेशन

अंतिम चरण में परिणाम प्रदर्शित करना और डाउनलोड विकल्प प्रदान करना शामिल है:

```python
if processed_text:
    # लंबे पाठ के लिए केवल एक सीमित पूर्वावलोकन दिखाएं
    MAX_PREVIEW_LINES = 250
    lines = processed_text.splitlines()
    if len(lines) > MAX_PREVIEW_LINES:
        # पहले और अंतिम कुछ लाइनें दिखाएँ...

    # आउटपुट प्रारूप के आधार पर टैब दिखाएँ
    if "HTML" in format_type:
        tab1, tab2 = st.tabs(["HTML पूर्वावलोकन", "परिणाम (HTML कोड)"])
        with tab1:
            components.html(preview_text, height=500, scrolling=True)
        with tab2:
            st.text_area("उत्पन्न HTML कोड:", preview_text, height=300)
    else:
        tab3_list = st.tabs(["परिणामी पाठ"])
        with tab3_list[0]:
            st.text_area("परिणाम:", preview_text, height=300)

    # डाउनलोड बटन प्रदान करें
    download_data = processed_text.encode('utf-8')
    st.download_button(
        label="परिणाम डाउनलोड करें",
        data=download_data,
        file_name="प्रतिस्थापन_परिणाम.html",
        mime="text/html"
    )
```

## समानांतर प्रसंस्करण कार्यान्वयन

बड़े पाठों के प्रदर्शन को अनुकूलित करने के लिए, अनुप्रयोग पायथन के `multiprocessing` मॉड्यूल का उपयोग करके समानांतर प्रसंस्करण प्रदान करता है।

### पाठ विभाजन

समानांतर प्रसंस्करण के लिए, पाठ को लाइनों में विभाजित किया जाता है और फिर प्रत्येक प्रक्रिया को लाइनों का एक बैच सौंपा जाता है:

```python
def parallel_process(
    text: str,
    num_processes: int,
    # अन्य पैरामीटर...
) -> str:
    # पहले देखें कि क्या समानांतर प्रसंस्करण उचित है
    if num_processes <= 1:
        return orchestrate_comprehensive_esperanto_text_replacement(
            # पैरामीटर...
        )

    # लाइनों में विभाजित करें
    lines = re.findall(r'.*?\n|.+$', text)
    num_lines = len(lines)

    # यदि केवल एक लाइन है, तो समानांतर प्रसंस्करण न करें
    if num_lines <= 1:
        return orchestrate_comprehensive_esperanto_text_replacement(
            # पैरामीटर...
        )

    # प्रत्येक प्रक्रिया के लिए लाइनों के रेंज निर्धारित करें
    lines_per_process = max(num_lines // num_processes, 1)
    ranges = [(i * lines_per_process, (i + 1) * lines_per_process) for i in range(num_processes)]
    ranges[-1] = (ranges[-1][0], num_lines)  # अंतिम प्रक्रिया को शेष सभी लाइनें दें
```

### प्रक्रियाओं का प्रबंधन

प्रक्रिया प्रबंधन और समन्वय पायथन की `multiprocessing.Pool` कक्षा का उपयोग करके किया जाता है:

```python
with multiprocessing.Pool(processes=num_processes) as pool:
    results = pool.starmap(
        process_segment,
        [
            (
                lines[start:end],
                placeholders_for_skipping_replacements,
                replacements_list_for_localized_string,
                placeholders_for_localized_replacement,
                replacements_final_list,
                replacements_list_for_2char,
                format_type
            )
            for (start, end) in ranges
        ]
    )
```

### परिणाम एकत्रीकरण

प्रत्येक प्रक्रिया अपने आवंटित लाइनों का प्रसंस्करण करती है, और परिणामों को अंतिम आउटपुट में संयोजित किया जाता है:

```python
return ''.join(results)
```

प्रत्येक प्रक्रिया `process_segment` फ़ंक्शन को निष्पादित करती है, जो लाइनों के अपने सेट को एक स्ट्रिंग में जोड़ता है और उसका प्रसंस्करण करता है:

```python
def process_segment(
    lines: List[str],
    # अन्य पैरामीटर...
) -> str:
    segment = ''.join(lines)
    result = orchestrate_comprehensive_esperanto_text_replacement(
        segment,
        # अन्य पैरामीटर...
    )
    return result
```

## प्रतिस्थापन तर्क

### सुरक्षित प्रतिस्थापन एल्गोरिथ्म

प्रतिस्थापन प्रक्रिया का केंद्र `safe_replace` फ़ंक्शन है, जो दो-चरण वाली प्रतिस्थापन रणनीति लागू करता है:

```python
def safe_replace(text: str, replacements: List[Tuple[str, str, str]]) -> str:
    """
    (old, new, placeholder) की एक सूची लेता है और प्रतिस्थापन को चरणों में लागू करता है:
    1. old → placeholder
    2. placeholder → new
    """
    valid_replacements = {}

    # पहला चरण: old → placeholder
    for old, new, placeholder in replacements:
        if old in text:
            text = text.replace(old, placeholder)
            valid_replacements[placeholder] = new

    # दूसरा चरण: placeholder → new
    for placeholder, new in valid_replacements.items():
        text = text.replace(placeholder, new)

    return text
```

यह दृष्टिकोण "पहले लंबे स्ट्रिंग्स को प्रतिस्थापित करें" के सिद्धांत को लागू करने के लिए महत्वपूर्ण है, क्योंकि प्रतिस्थापनों को प्लेसहोल्डर्स के माध्यम से मध्यवर्ती चरण में स्टोर किया जाता है।

### वर्ण रूपांतरण

अनुप्रयोग के कई स्थानों पर एस्पेरांतो विशेष वर्ण रूपांतरण उपयोग किए जाते हैं:

```python
def replace_esperanto_chars(text, char_dict: Dict[str, str]) -> str:
    """
    शब्दकोश के अनुसार स्ट्रिंग में वर्णों को प्रतिस्थापित करें
    """
    for original_char, converted_char in char_dict.items():
        text = text.replace(original_char, converted_char)
    return text
```

इस फ़ंक्शन को विभिन्न वर्ण मैपिंग शब्दकोशों के साथ कॉल किया जाता है जैसे `x_to_circumflex`, `hat_to_circumflex`, आदि।

### प्लेसहोल्डर उपयोग

प्लेसहोल्डर्स विभिन्न उद्देश्यों के लिए उपयोग किए जाते हैं:

1. **सुरक्षित प्रतिस्थापन के लिए**: यह सुनिश्चित करने के लिए कि पहले लंबे प्रतिस्थापनों को प्राथमिकता दी जाती है
2. **पाठ के कुछ हिस्सों को प्रतिस्थापन से छोड़ने के लिए**: %...% से घिरे भाग
3. **पाठ के स्थानीय प्रतिस्थापन के लिए**: @...@ से घिरे भाग

प्लेसहोल्डर्स को फाइलों से पढ़ा जाता है और विभिन्न प्रतिस्थापन सूचियों में उपयोग किया जाता है।

### % और @ के साथ विशेष प्रतिस्थापन

अनुप्रयोग % (प्रतिस्थापन से छोड़ना) और @ (स्थानीय प्रतिस्थापन) के लिए रेगुलर एक्सप्रेशन का उपयोग करता है:

```python
# % के लिए नियमित अभिव्यक्ति
PERCENT_PATTERN = re.compile(r'%(.{1,50}?)%')

def find_percent_enclosed_strings_for_skipping_replacement(text: str) -> List[str]:
    """'%foo%' पैटर्न की सभी स्ट्रिंग्स निकालें, 50 वर्णों तक सीमित"""
    matches = []
    used_indices = set()
    for match in PERCENT_PATTERN.finditer(text):
        start, end = match.span()
        if start not in used_indices and end-2 not in used_indices:
            matches.append(match.group(1))
            used_indices.update(range(start, end))
    return matches

# @ के लिए समान तर्क
AT_PATTERN = re.compile(r'@(.{1,18}?)@')
# ...
```

ये पैटर्न स्किपिंग और स्थानीय प्रतिस्थापन के लिए पाठ के विशेष भागों को पहचानने के लिए उपयोग किए जाते हैं।

## JSON जनरेशन

### CSV पार्सिंग और प्रसंस्करण

JSON जनरेटर पेज CSV फाइलों को पार्स करता है जिनमें एस्पेरांतो मूल शब्द और उनके अनुवाद (कान्जी/हिंदी) शामिल हैं:

```python
if csv_choice == "CSV अपलोड करें":
    uploaded_file = st.file_uploader("CSV फ़ाइल चुनें", type=['csv'])
    if uploaded_file is not None:
        file_contents = uploaded_file.read().decode("utf-8")
        converted_text = convert_to_circumflex(file_contents)
        csv_buffer = StringIO(converted_text)
        CSV_data_imported = pd.read_csv(csv_buffer, encoding="utf-8", usecols=[0, 1])
```

CSV डेटा इम्पोर्ट करने के बाद, यह एक अस्थायी प्रतिस्थापन शब्दकोश में प्रसंस्करित किया जाता है:

```python
for *, (E*root, hanzi_or_meaning) in CSV_data_imported.iterrows():
    if pd.notna(E_root) and pd.notna(hanzi_or_meaning) \
       and '#' not in E_root and (E_root != '') and (hanzi_or_meaning != ''):
        temporary_replacements_dict[E_root] = [
            output_format(E_root, hanzi_or_meaning, format_type, char_widths_dict),
            len(E_root)
        ]
```

### मूल शब्द निष्कर्षण

JSON जनरेशन प्रक्रिया एस्पेरांतो भाषा की संरचना का उपयोग करके मूल शब्दों के प्रसंस्करण पर निर्भर करती है। यह विभिन्न शब्दांशों को पहचानता है:

1. **मूल शब्द**: एस्पेरांतो शब्दों के मूल भाग
2. **प्रत्यय**: जैसे -o (नाउन), -a (विशेषण), -e (क्रिया विशेषण), -i (इनफिनिटिव क्रिया)
3. **क्रिया रूप**: -as (वर्तमान), -is (भूतकाल), -os (भविष्य), -us (शर्तात्मक)
4. **विशेष दो-वर्ण मूल**: उपसर्ग, प्रत्यय, और स्वतंत्र रूप

इन शब्दांशों को पहचानने और प्रसंस्करित करने के लिए जटिल तर्क लागू किया जाता है:

```python
verb_suffix_2l = {
    'as':'as', 'is':'is', 'os':'os', 'us':'us','at':'at','it':'it','ot':'ot',
    'ad':'ad','iĝ':'iĝ','ig':'ig','ant':'ant','int':'int','ont':'ont'
}

# उपसर्ग और प्रत्यय के लिए अलग संग्रह
suffix_2char_roots = ['ad', 'ag', 'am', 'ar', 'as', ...]
prefix_2char_roots = ['al', 'am', 'av', 'bo', 'di', ...]
standalone_2char_roots = ['al', 'ci', 'da', 'de', 'di', ...]
```

### प्रतिस्थापन नियम निर्माण

प्रतिस्थापन नियमों को कई चरणों में बनाया जाता है:

1. **अस्थायी प्रतिस्थापन डिक्शनरी**: CSV से आयातित मूल डेटा
2. **प्री-प्रतिस्थापन डिक्शनरी**: एस्पेरांतो मूल शब्दों के साथ जुड़ी भाषण की भागों के आधार पर बनाया गया
3. **प्राथमिकता समायोजन**: प्रत्ययों (जैसे -an, -on) और क्रिया रूपों के लिए विशेष समायोजन
4. **कस्टम स्टेमिंग सेटिंग्स**: उपयोगकर्ता-परिभाषित नियमों को लागू करना
5. **अंतिम संयुक्त सूची**: तीन मुख्य प्रतिस्थापन सूचियों में संयोजित किया गया

एक महत्वपूर्ण भाग यह है कि शब्दों को उनकी लंबाई के अनुसार प्रतिस्थापन प्राथमिकता दी जाती है, जिससे सुनिश्चित होता है कि लंबे शब्द पहले प्रतिस्थापित होते हैं:

```python
# प्रतिस्थापन प्राथमिकता सेट करें
pre_replacements_dict_2[i.replace('/', '')] = [
    j[0].replace("</rt></ruby>","%%%").replace('/', '').replace("%%%","</rt></ruby>"),
    j[1],
    len(i.replace('/', ''))*10000  # लंबाई * 10000 प्राथमिकता बनाता है
]
```

### JSON संरचना

अंतिम JSON फाइल में तीन प्रमुख घटक शामिल हैं:

1. **वैश्विक प्रतिस्थापन सूची**: सामान्य एस्पेरांतो शब्दों के लिए
2. **दो-वर्ण मूल शब्द प्रतिस्थापन सूची**: उपसर्ग, प्रत्यय, और स्वतंत्र दो-वर्ण मूल शब्दों के लिए
3. **स्थानीय प्रतिस्थापन सूची**: @...@ पैटर्न से घिरे स्थानीय प्रतिस्थापन के लिए

प्रत्येक प्रतिस्थापन सूची में (old_text, new_text, placeholder) ट्रिपल्स शामिल होते हैं।

```python
combined_data = {}
combined_data["全域替换用のリスト(列表)型配列(replacements_final_list)"] = replacements_final_list
combined_data["二文字词根替换用のリスト(列表)型配列(replacements_list_for_2char)"] = replacements_list_for_2char
combined_data["局部文字替换用のリスト(列表)型配列(replacements_list_for_localized_string)"] = replacements_list_for_localized_string
```

## उन्नत विशेषताएँ

### कस्टम आउटपुट फॉर्मेट

अनुप्रयोग विभिन्न आउटपुट फॉर्मेट का समर्थन करता है, जैसे:

```python
options = {
    'HTML格式_Ruby文字_大小调整': 'HTML格式_Ruby文字_大小调整',
    'HTML格式_Ruby文字_大小调整_汉字替换': 'HTML格式_Ruby文字_大小调整_汉字替换',
    'HTML格式': 'HTML格式',
    'HTML格式_漢字替换': 'HTML格式_漢字替换',
    '括弧(号)格式': '括弧(号)格式',
    '括弧(号)格式_漢字替换': '括弧(号)格式_漢字替换',
    '替换后文字列のみ(仅)保留(简单替换)': '替换后文字列のみ(仅)保留(简单替换)'
}
```

प्रत्येक फॉर्मेट `output_format` फ़ंक्शन के माध्यम से लागू किया जाता है:

```python
def output_format(main_text, ruby_content, format_type, char_widths_dict):
    """
    एस्पेरांतो मूल शब्द और रूबी सामग्री (अनुवाद) को निर्दिष्ट प्रारूप में स्वरूपित करता है
    """
    if format_type == 'HTML格式_Ruby文字_大小调整':
        # रूबी और मुख्य पाठ का अनुपात आधारित समायोजन
        width_ruby = measure_text_width_Arial16(ruby_content, char_widths_dict)
        width_main = measure_text_width_Arial16(main_text, char_widths_dict)
        ratio_1 = width_ruby / width_main

        # अनुपात के आधार पर विभिन्न CSS कक्षाएँ लागू करें
        if ratio_1 > 6:
            return f'<ruby>{main_text}<rt class="XXXS_S">{insert_br_at_third_width(ruby_content, char_widths_dict)}</rt></ruby>'
        elif ratio_1 > (9/3):
            return f'<ruby>{main_text}<rt class="XXS_S">{insert_br_at_half_width(ruby_content, char_widths_dict)}</rt></ruby>'
        # अन्य अनुपात केस...
```

### रूबी एनोटेशन

रूबी एनोटेशन HTML के उपयोग से लागू की जाती है, जहां मुख्य पाठ पर छोटे फ़ॉन्ट में अर्थ दिखाया जाता है:

```html
<ruby>esperanto<rt>एस्पेरान्तो</rt></ruby>
```

HTML हेडर और फुटर जोड़ने का विशेष फ़ंक्शन है:

```python
def apply_ruby_html_header_and_footer(processed_text: str, format_type: str) -> str:
    """
    चुने गए फॉर्मेट के आधार पर HTML हेडर/फुटर और स्टाइल जोड़ें
    """
    if format_type in ('HTML格式_Ruby文字_大小调整','HTML格式_Ruby文字_大小调整_汉字替换'):
        ruby_style_head="""<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>大多数の环境中で正常に运行するRuby显示功能</title>
    <style>
    /* विस्तृत CSS स्टाइल्स */
    </style>
  </head>
  <body>
  <p class="text-M_M">
"""
        ruby_style_tail = "</p></body></html>"
    # अन्य प्रारूपों के लिए...
```

### वर्ण चौड़ाई समायोजन

अनुप्रयोग वर्ण चौड़ाई मापन के आधार पर स्वचालित ब्रेकिंग और समायोजन प्रदान करता है:

```python
def measure_text_width_Arial16(text, char_widths_dict: Dict[str, int]) -> int:
    """
    JSON से लोड किए गए वर्ण चौड़ाई डिक्शनरी का उपयोग करके टेक्स्ट की कुल चौड़ाई मापें
    """
    total_width = 0
    for ch in text:
        char_width = char_widths_dict.get(ch, 8)
        total_width += char_width
    return total_width

def insert_br_at_half_width(text, char_widths_dict: Dict[str, int]) -> str:
    """
    लगभग आधी चौड़ाई पर <br> डालकर टेक्स्ट विभाजित करें
    """
    total_width = measure_text_width_Arial16(text, char_widths_dict)
    half_width = total_width / 2
    current_width = 0
    insert_index = None

    for i, ch in enumerate(text):
        char_width = char_widths_dict.get(ch, 8)
        current_width += char_width
        if current_width >= half_width:
            insert_index = i + 1
            break

    if insert_index is not None:
        result = text[:insert_index] + "<br>" + text[insert_index:]
    else:
        result = text
    return result
```

## प्रदर्शन अनुकूलन

### कैशिंग तकनीक

अनुप्रयोग स्ट्रीमलिट के `@st.cache_data` डेकोरेटर का उपयोग करके JSON लोडिंग को अनुकूलित करता है:

```python
@st.cache_data
def load_replacements_lists(json_path: str) -> Tuple[List, List, List]:
    """
    JSON फाइल लोड करें और प्रतिस्थापन सूचियों का एक टपल वापस करें
    """
    with open(json_path, 'r', encoding='utf-8') as f:
        data = json.load(f)
    replacements_final_list = data.get(
        "全域替换用のリスト(列表)型配列(replacements_final_list)", []
    )
    # अन्य सूचियां...
    return (
        replacements_final_list,
        replacements_list_for_localized_string,
        replacements_list_for_2char,
    )
```

स्ट्रीमलिट का `@st.cache_data` डेकोरेटर पहले कॉल के परिणाम को कैश करता है, जिससे बार-बार लोडिंग से बचा जा सकता है, बड़ी JSON फाइलों (50MB तक) के साथ महत्वपूर्ण प्रदर्शन लाभ प्रदान करता है।

### समानांतर प्रसंस्करण के प्रभाव

समानांतर प्रसंस्करण बड़े पाठों के लिए महत्वपूर्ण प्रदर्शन लाभ प्रदान कर सकता है। उपयोगकर्ता इंटरफेस में, इसे कॉन्फ़िगर किया जा सकता है:

```python
use_parallel = st.checkbox("पैरलल प्रोसेसिंग का उपयोग करें", value=False)
num_processes = st.number_input(
    "समकालिक प्रोसेस की संख्या",
    min_value=2, max_value=4, value=4, step=1
)
```

हालांकि, समानांतर प्रसंस्करण केवल लंबे पाठों के लिए उपयोग किया जाना चाहिए:

```python
if num_processes <= 1:
    # सिंगल-थ्रेडेड प्रोसेसिंग का उपयोग करें
    return orchestrate_comprehensive_esperanto_text_replacement(...)

# लाइनों को गिनें
lines = re.findall(r'.*?\n|.+$', text)
num_lines = len(lines)
if num_lines <= 1:
    # लाइनों की संख्या कम है, सिंगल-थ्रेडेड प्रोसेसिंग का उपयोग करें
    return orchestrate_comprehensive_esperanto_text_replacement(...)
```

समानांतर प्रसंस्करण से रन-टाइम में लंबे पाठों के प्रसंस्करण समय को काफी कम किया जा सकता है।

### मेमोरी प्रबंधन

अनुप्रयोग में बड़े पाठों के लिए मेमोरी प्रबंधन के कई तंत्र हैं:

1. **आंशिक पूर्वावलोकन**: केवल प्रसंस्करित पाठ की पहले 250 लाइनें दिखाता है
2. **स्ट्रीमलिट कैशिंग**: मेमोरी का कुशल उपयोग करने के लिए
3. **प्रक्रिया प्रबंधन**: `multiprocessing` मॉड्यूल का उपयोग करके प्रक्रियाओं को प्रबंधित करना

```python
if len(lines) > MAX_PREVIEW_LINES:
    first_part = lines[:247]
    last_part = lines[-3:]
    preview_text = "\n".join(first_part) + "\n...\n" + "\n".join(last_part)
    st.warning(
        f"पाठ बहुत लंबा है (कुल {len(lines)} पंक्तियाँ)। "
        "केवल आंशिक पूर्वावलोकन दिखाया जा रहा है (प्रथम 247 पंक्तियाँ और अंतिम 3 पंक्तियाँ)।"
    )
```

multiprocessing मॉड्यूल के साथ कार्य करते समय, अनुप्रयोग "spawn" मोड का उपयोग करता है ताकि बेहतर मेमोरी प्रबंधन सुनिश्चित हो:

```python
try:
    multiprocessing.set_start_method("spawn")
except RuntimeError:
    pass  # पहले से ही सेट है तो इग्नोर करें
```

इस अनुप्रयोग के प्रोग्रामिंग डिजाइन में कई परिष्कृत विशेषताएँ हैं जो इसे एक मजबूत और कुशल टूल बनाती हैं। यह सेक्शन प्रत्येक भाग के तकनीकी विवरण का अवलोकन प्रदान करता है, और अनुप्रयोग के तकनीकी कार्यान्वयन को समझने में मदद करता है।
