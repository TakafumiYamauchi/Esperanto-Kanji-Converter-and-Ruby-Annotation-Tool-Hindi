# एस्पेरांतो पाठ प्रतिस्थापन एप्लिकेशन की तकनीकी संरचना का विश्लेषण

## परिचय

यह दस्तावेज़ एस्पेरांतो पाठ प्रतिस्थापन एप्लिकेशन के अंतर्निहित कोड संरचना और कार्यप्रणाली का विस्तृत विश्लेषण प्रदान करता है। एप्लिकेशन में चार मुख्य पायथन फाइलें शामिल हैं:

1. `main.py` - मुख्य एप्लिकेशन फाइल जो Streamlit इंटरफेस प्रदान करती है
2. `एस्पेरान्तो पाठ को स्ट्रिंग (कांजी) से प्रतिस्थापित करने हेतु JSON फ़ाइल बनाने का पृष्ठ.py` - JSON जनरेटर पेज
3. `esp_text_replacement_module.py` - टेक्स्ट प्रतिस्थापन यूटिलिटी मॉड्यूल
4. `esp_replacement_json_make_module.py` - JSON फाइल जनरेशन के लिए उपयोगिताएँ

इस दस्तावेज़ में हम प्रत्येक फाइल की संरचना, मुख्य फंक्शन, और उनके पारस्परिक संबंधों पर चर्चा करेंगे ताकि आप इस एप्लिकेशन के आर्किटेक्चर को गहराई से समझ सकें।

## 1. उच्च स्तरीय आर्किटेक्चर ओवरव्यू

### 1.1 एप्लिकेशन संरचना

एप्लिकेशन एक Streamlit वेब ऐप है जो मूलतः दो पेज प्रदान करता है:

- **मुख्य पेज** (`main.py`): एस्पेरांतो पाठ को काञ्जी (चीनी अक्षर) या अन्य भाषाओं के समकक्ष में परिवर्तित करता है, HTML रूबी एनोटेशन या अन्य प्रारूपों में।
- **JSON जनरेटर पेज** (`एस्पेरान्तो पाठ को...`): उपयोगकर्ताओं को प्रतिस्थापन नियमों वाली अपनी JSON फाइलें बनाने की अनुमति देता है।

### 1.2 डेटा प्रवाह

एप्लिकेशन में डेटा प्रवाह इस प्रकार है:

1. उपयोगकर्ता या तो डिफॉल्ट JSON प्रतिस्थापन फाइल का उपयोग करता है या अपनी खुद की अपलोड करता है
2. उपयोगकर्ता एस्पेरांतो पाठ इनपुट करता है (टाइप करके या फाइल अपलोड करके)
3. पाठ को एक टेक्स्ट प्रोसेसिंग पाइपलाइन (निम्न में से कई चरणों में) से गुजारा जाता है:
   - एस्पेरांतो विशेष अक्षरों को मानकीकृत करना
   - प्रतिस्थापन से बचाए जाने वाले क्षेत्रों को चिह्नित करना (प्लेसहोल्डर्स का उपयोग करके)
   - स्थानीय प्रतिस्थापन लागू करना
   - वैश्विक प्रतिस्थापन लागू करना
   - विशेष 2-अक्षर प्रतिस्थापन लागू करना
   - प्लेसहोल्डर्स की पुनर्स्थापना
4. परिवर्तित पाठ HTML या अन्य प्रारूपों में प्रदर्शित किया जाता है और डाउनलोड के लिए उपलब्ध कराया जाता है

### 1.3 मॉड्यूल के बीच संबंध

मॉड्यूल के बीच संबंध इस प्रकार हैं:

- `main.py` टेक्स्ट प्रतिस्थापन के लिए `esp_text_replacement_module.py` से फंक्शन आयात करता है
- `एस्पेरान्तो पाठ को...` फाइल दोनों `esp_text_replacement_module.py` और `esp_replacement_json_make_module.py` से फंक्शंस आयात करती है
- दोनों यूटिलिटी मॉड्यूल कुछ समान फंक्शन्स को परिभाषित करते हैं (कुछ दोहराव के साथ) लेकिन अलग-अलग उद्देश्यों के लिए

अब हम प्रत्येक फाइल के अंदर गहराई से जाएंगे और उनकी प्रमुख विशेषताओं पर चर्चा करेंगे।

## 2. main.py - मुख्य एप्लिकेशन फाइल

`main.py` एप्लिकेशन की मुख्य फाइल है जो ग्राफिकल इंटरफेस और टेक्स्ट प्रतिस्थापन की मुख्य कार्यक्षमता प्रदान करती है।

### 2.1 आयात और सेटअप

```python
import streamlit as st
import re
import io
import json
import pandas as pd  # यदि आवश्यक हो
from typing import List, Dict, Tuple, Optional
import streamlit.components.v1 as components
import multiprocessing

# multiprocessing के लिए 'spawn' मोड सेट करना
try:
    multiprocessing.set_start_method("spawn")
except RuntimeError:
    pass  # यदि विधि पहले से सेट है तो इसे अनदेखा करें

# esp_text_replacement_module से फंक्शन आयात
from esp_text_replacement_module import (
    x_to_circumflex,
    x_to_hat,
    hat_to_circumflex,
    circumflex_to_hat,
    replace_esperanto_chars,
    import_placeholders,
    orchestrate_comprehensive_esperanto_text_replacement,
    parallel_process,
    apply_ruby_html_header_and_footer
)
```

महत्वपूर्ण बिंदु:
- **multiprocessing**: एप्लिकेशन समानांतर प्रोसेसिंग के लिए पायथन के `multiprocessing` मॉड्यूल का उपयोग करता है (`spawn` मोड में)
- **टाइप हिंट्स**: कोड टाइप हिंट्स (`List`, `Dict`, आदि) का उपयोग करता है जो पायथन टाइप सिस्टम का सुझाव देता है
- **esp_text_replacement_module**: एप्लिकेशन एस्पेरांतो अक्षरों और टेक्स्ट प्रतिस्थापन से संबंधित विभिन्न फंक्शन आयात करता है

### 2.2 JSON फाइल लोडिंग

```python
@st.cache_data
def load_replacements_lists(json_path: str) -> Tuple[List, List, List]:
    """
    JSON फाइल लोड करता है और 3 लिस्ट्स का टपल रिटर्न करता है:
    1) replacements_final_list
    2) replacements_list_for_localized_string
    3) replacements_list_for_2char
    """
    with open(json_path, 'r', encoding='utf-8') as f:
        data = json.load(f)
    replacements_final_list = data.get(
        "全域替换用のリスト(列表)型配列(replacements_final_list)", []
    )
    replacements_list_for_localized_string = data.get(
        "局部文字替换用のリスト(列表)型配列(replacements_list_for_localized_string)", []
    )
    replacements_list_for_2char = data.get(
        "二文字词根替换用のリスト(列表)型配列(replacements_list_for_2char)", []
    )
    return (
        replacements_final_list,
        replacements_list_for_localized_string,
        replacements_list_for_2char,
    )
```

महत्वपूर्ण बिंदु:
- **@st.cache_data**: यह Streamlit डेकोरेटर फंक्शन के परिणामों को कैश करता है, जिससे बड़ी JSON फाइलों को बार-बार लोड करने से बचा जा सकता है
- **तीन प्रकार के प्रतिस्थापन**: फंक्शन तीन अलग प्रतिस्थापन सूचियां रिटर्न करता है, प्रत्येक का अपना विशिष्ट उद्देश्य है:
  1. `replacements_final_list` - वैश्विक प्रतिस्थापन के लिए
  2. `replacements_list_for_localized_string` - स्थानीय प्रतिस्थापन के लिए (@...@ के अंदर)
  3. `replacements_list_for_2char` - विशेष रूप से 2-अक्षर वाले एस्पेरांतो रूट्स के लिए

### 2.3 Streamlit पेज सेटअप

```python
st.set_page_config(
    page_title="एस्पेरांतो पाठ में कैरेक्टर (कान्जी) बदलने का उपकरण",
    layout="wide"
)
# शीर्षक (हिंदी में GUI प्रदर्शन के लिए)
st.title("एस्पेरांतो पाठ को कान्जी या HTML एनोटेशन से बदलना (विस्तारित संस्करण)")
st.write("---")
```

यह खंड Streamlit पेज के लेआउट और शीर्षक को कॉन्फ़िगर करता है।

### 2.4 JSON फाइल का लोडिंग और प्लेसहोल्डर का आयात

```python
# JSON फाइल लोड करना
json_options = ["デフォルトを使用する", "アップロードする"]
selected_option = st.radio(
    "JSON फ़ाइल के साथ कैसे कार्य करना है? (रिप्लेसमेंट JSON फ़ाइल को लोड करने के लिए)",
    json_options,
    format_func=lambda x: "डिफ़ॉल्ट JSON का उपयोग करें" if x == "デフォルトを使用する" else "फ़ाइल अपलोड करें"
)

# प्लेसहोल्डर्स लोड करना
placeholders_for_skipping_replacements: List[str] = import_placeholders(
    './Appの运行に使用する各类文件/占位符(placeholders)_%1854%-%4934%_文字列替换skip用.txt'
)
placeholders_for_localized_replacement: List[str] = import_placeholders(
    './Appの运行に使用する各类文件/占位符(placeholders)_@5134@-@9728@_局部文字列替换结果捕捉用.txt'
)
```

महत्वपूर्ण बिंदु:
- **JSON विकल्प**: उपयोगकर्ता या तो डिफॉल्ट JSON फाइल का उपयोग कर सकता है या अपनी खुद की अपलोड कर सकता है
- **प्लेसहोल्डर्स**: दो प्लेसहोल्डर फाइलें लोड की जाती हैं:
  1. `placeholders_for_skipping_replacements` - प्रतिस्थापन से बचे हुए पाठ (%...%) को अस्थायी रूप से स्टोर करने के लिए
  2. `placeholders_for_localized_replacement` - स्थानीय प्रतिस्थापन (@...@) के लिए

### 2.5 टेक्स्ट प्रोसेसिंग फॉर्म और प्रतिस्थापन कार्यान्वयन

```python
with st.form(key='profile_form'):
    # टेक्स्ट इनपुट, लेटर प्रकार चयन, आदि...

    submit_btn = st.form_submit_button('सबमिट')
    cancel_btn = st.form_submit_button("रद्द करें")

    if cancel_btn:
        st.warning("कार्रवाई रद्द कर दी गई।")
        st.stop()

    if submit_btn:
        st.session_state["text0_value"] = text0
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
                text=text0,
                placeholders_for_skipping_replacements=placeholders_for_skipping_replacements,
                replacements_list_for_localized_string=replacements_list_for_localized_string,
                placeholders_for_localized_replacement=placeholders_for_localized_replacement,
                replacements_final_list=replacements_final_list,
                replacements_list_for_2char=replacements_list_for_2char,
                format_type=format_type
            )

        # विशेष अक्षर प्रतिस्थापन (x_to_circumflex, आदि) लागू करें
        if letter_type == '上付き文字':
            processed_text = replace_esperanto_chars(processed_text, x_to_circumflex)
            processed_text = replace_esperanto_chars(processed_text, hat_to_circumflex)
        elif letter_type == '^形式':
            processed_text = replace_esperanto_chars(processed_text, x_to_hat)
            processed_text = replace_esperanto_chars(processed_text, circumflex_to_hat)

        processed_text = apply_ruby_html_header_and_footer(processed_text, format_type)
```

महत्वपूर्ण बिंदु:
- **समानांतर/एकल थ्रेड प्रोसेसिंग**: उपयोगकर्ता के चयन के आधार पर, एप्लिकेशन या तो `parallel_process` या `orchestrate_comprehensive_esperanto_text_replacement` का उपयोग करता है
- **एस्पेरांतो अक्षर प्रतिस्थापन**: उपयोगकर्ता द्वारा चुने गए दिखावट के अनुसार एस्पेरांतो विशेष अक्षरों का प्रतिस्थापन किया जाता है
- **HTML हेडर/फुटर अप्लाई करना**: आउटपुट प्रारूप के आधार पर उपयुक्त HTML हेडर और फुटर जोड़े जाते हैं

### 2.6 परिणाम प्रदर्शन और डाउनलोड

```python
if processed_text:
    MAX_PREVIEW_LINES = 250
    lines = processed_text.splitlines()
    if len(lines) > MAX_PREVIEW_LINES:
        # बहुत बड़े पाठ के लिए आंशिक प्रीव्यू...
    else:
        preview_text = processed_text

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

    download_data = processed_text.encode('utf-8')
    st.download_button(
        label="परिणाम डाउनलोड करें",
        data=download_data,
        file_name="प्रतिस्थापन_परिणाम.html",
        mime="text/html"
    )
```

महत्वपूर्ण बिंदु:
- **बड़े पाठ के लिए आंशिक प्रीव्यू**: अत्यधिक लंबे पाठ के लिए, एप्लिकेशन केवल पहली 247 और अंतिम 3 लाइनें दिखाता है
- **HTML प्रीव्यू**: HTML प्रारूप के लिए, एप्लिकेशन दो टैब प्रदान करता है—एक रेंडर्ड HTML के लिए और एक सोर्स कोड के लिए
- **डाउनलोड बटन**: उपयोगकर्ता प्रोसेस्ड पाठ को HTML फाइल के रूप में डाउनलोड कर सकता है

## 3. esp_text_replacement_module.py - टेक्स्ट प्रतिस्थापन की मुख्य संरचना

`esp_text_replacement_module.py` एस्पेरांतो पाठ प्रतिस्थापन और प्रसंस्करण के लिए मुख्य कार्यात्मकता प्रदान करता है।

### 3.1 एस्पेरांतो अक्षर मैपिंग

```python
# ================================
# 1) एस्पेरांतो अक्षर रूपांतरण के लिए शब्दकोष
# ================================
x_to_circumflex = {
    'cx': 'ĉ', 'gx': 'ĝ', 'hx': 'ĥ', 'jx': 'ĵ', 'sx': 'ŝ', 'ux': 'ŭ',
    'Cx': 'Ĉ', 'Gx': 'Ĝ', 'Hx': 'Ĥ', 'Jx': 'Ĵ', 'Sx': 'Ŝ', 'Ux': 'Ŭ'
}
circumflex_to_x = {...}  # उलटा मैपिंग
x_to_hat = {...}  # 'cx' -> 'c^', आदि
hat_to_x = {...}  # 'c^' -> 'cx', आदि
hat_to_circumflex = {...}  # 'c^' -> 'ĉ', आदि
circumflex_to_hat = {...}  # 'ĉ' -> 'c^', आदि
```

यह खंड एस्पेरांतो विशेष अक्षरों के विभिन्न प्रतिनिधित्वों के बीच मैपिंग परिभाषित करता है:
- **x-सिस्टम**: 'cx', 'gx', आदि (आसान टाइपिंग के लिए)
- **हैट (^) सिस्टम**: 'c^', 'g^', आदि
- **सर्कमफ्लेक्स सिस्टम**: 'ĉ', 'ĝ', आदि (एस्पेरांतो के मानक अक्षर)

### 3.2 मूलभूत अक्षर प्रतिस्थापन फंक्शन्स

```python
def replace_esperanto_chars(text, char_dict: Dict[str, str]) -> str:
    # char_dict में निहित जोड़े (original_char, converted_char) के लिए
    # text.replace() करें
    for original_char, converted_char in char_dict.items():
        text = text.replace(original_char, converted_char)
    return text

def convert_to_circumflex(text: str) -> str:
    """
    पाठ को सर्कमफ्लेक्स प्रारूप (ĉ, ĝ, ĥ, ĵ, ŝ, ŭ आदि) में एकीकृत करता है।
    1. hat_to_circumflex: c^ → ĉ
    2. x_to_circumflex: cx → ĉ
    """
    text = replace_esperanto_chars(text, hat_to_circumflex)
    text = replace_esperanto_chars(text, x_to_circumflex)
    return text

def unify_halfwidth_spaces(text: str) -> str:
    """
    फुलविड्थ स्पेस (U+3000) को अपरिवर्तित छोड़ते हुए, हाफविड्थ स्पेस और विज़ुअली मिलते-जुलते
    स्पेस कैरेक्टर्स को ASCII हाफविड्थ स्पेस (U+0020) में एकीकृत करता है।
    """
    pattern = r"[\u00A0\u2002\u2003\u2004\u2005\u2006\u2007\u2008\u2009\u200A]"
    return re.sub(pattern, " ", text)
```

ये फंक्शन एस्पेरांतो अक्षरों के बीच रूपांतरण प्रदान करते हैं और स्पेस कैरेक्टर्स को एकीकृत करते हैं।

### 3.3 प्लेसहोल्डर और सुरक्षित प्रतिस्थापन

```python
def safe_replace(text: str, replacements: List[Tuple[str, str, str]]) -> str:
    """
    (old, new, placeholder) की सूची प्राप्त करता है,
    text में old → placeholder → new के चरणबद्ध प्रतिस्थापन को करता है।
    """
    valid_replacements = {}
    # पहले old→placeholder
    for old, new, placeholder in replacements:
        if old in text:
            text = text.replace(old, placeholder)
            valid_replacements[placeholder] = new
    # फिर placeholder→new
    for placeholder, new in valid_replacements.items():
        text = text.replace(placeholder, new)
    return text

def import_placeholders(filename: str) -> List[str]:
    """
    प्लेसहोल्डर्स को रेखा-दर-रेखा पढ़ने वाला सरल फंक्शन
    """
    with open(filename, 'r') as file:
        placeholders = [line.strip() for line in file if line.strip()]
    return placeholders
```

इन फंक्शन्स का उपयोग प्रतिस्थापन प्रक्रिया में किया जाता है:
- **safe_replace**: यह फंक्शन एक सुरक्षित प्रतिस्थापन तंत्र का उपयोग करता है जो पहले मूल टेक्स्ट को प्लेसहोल्डर से बदलता है और फिर प्लेसहोल्डर को अंतिम वैल्यू से बदलता है, इस प्रकार रिकर्सिव प्रतिस्थापन से बचता है
- **import_placeholders**: यह फंक्शन फाइल से प्लेसहोल्डर स्ट्रिंग्स को लाइन-बाय-लाइन पढ़ता है

### 3.4 विशेष पैटर्न की पहचान और प्रतिस्थापन सूचियाँ बनाना

```python
# '%' से घिरे क्षेत्रों को प्रतिस्थापन से छोड़ने के लिए रेगेक्स
PERCENT_PATTERN = re.compile(r'%(.{1,50}?)%')

def find_percent_enclosed_strings_for_skipping_replacement(text: str) -> List[str]:
    """'%foo%' के सभी प्रारूपों को निकालता है। 50 अक्षरों तक सीमित।"""
    # ...

def create_replacements_list_for_intact_parts(text: str, placeholders: List[str]) -> List[Tuple[str, str]]:
    """
    '%xxx%' से घिरे क्षेत्रों का पता लगाता है,
    ( '%xxx%', placeholder ) के प्रारूप में प्रत्येक को मैप करने वाली सूची बनाता है
    """
    # ...

# '@' से घिरे क्षेत्रों को स्थानीय प्रतिस्थापन के लिए रेगेक्स
AT_PATTERN = re.compile(r'@(.{1,18}?)@')

def find_at_enclosed_strings_for_localized_replacement(text: str) -> List[str]:
    """'@foo@' के सभी प्रारूपों को निकालता है। 18 अक्षरों तक सीमित।"""
    # ...

def create_replacements_list_for_localized_replacement(text, placeholders: List[str],
                                                       replacements_list_for_localized_string: List[Tuple[str, str, str]]
                                                       ) -> List[List[str]]:
    """
    '@xxx@' से घिरे क्षेत्रों का पता लगाता है,
    उनके अंदर के पाठ 'xxx' को replacements_list_for_localized_string के साथ बदलता है
    और परिणाम को प्लेसहोल्डर से बदलता है।
    """
    matches = find_at_enclosed_strings_for_localized_replacement(text)
    tmp_list = []
    for i, match in enumerate(matches):
        if i < len(placeholders):
            replaced_match = safe_replace(match, replacements_list_for_localized_string)
            tmp_list.append([f"@{match}@", placeholders[i], replaced_match])
        else:
            break
    return tmp_list

### 3.5 मुख्य प्रतिस्थापन ऑर्केस्ट्रेशन फंक्शन

```python
def orchestrate_comprehensive_esperanto_text_replacement(
    text,
    placeholders_for_skipping_replacements: List[str],
    replacements_list_for_localized_string: List[Tuple[str, str, str]],
    placeholders_for_localized_replacement: List[str],
    replacements_final_list: List[Tuple[str, str, str]],
    replacements_list_for_2char: List[Tuple[str, str, str]],
    format_type: str
) -> str:
    """
    एस्पेरांतो पाठ को विभिन्न रूपांतरण नियमों के अनुसार व्यापक रूप से प्रतिस्थापित करने वाला मुख्य फंक्शन।
    1) स्पेस का मानकीकरण → 2) एस्पेरांतो अक्षरों (ĉ आदि) को सर्कमफ्लेक्स प्रारूप में एकीकृत करना
    3) % से घिरे हिस्सों को छोड़ना
    4) @ से घिरे हिस्सों को स्थानीय रूप से प्रतिस्थापित करना
    5) वैश्विक प्रतिस्थापन
    6) 2-अक्षर शब्दमूलों को 2 बार प्रतिस्थापित करना
    7) प्लेसहोल्डर पुनर्स्थापना
    8) HTML प्रारूप निर्दिष्ट होने पर अतिरिक्त फॉर्मेटिंग
    """

    # 1, 2) स्पेस का मानकीकरण + एस्पेरांतो अक्षरों को सर्कमफ्लेक्स में परिवर्तन
    text = unify_halfwidth_spaces(text)
    text = convert_to_circumflex(text)

    # 3) %...% स्किप क्षेत्रों को अस्थायी रूप से प्रतिस्थापित करना
    replacements_list_for_intact_parts = create_replacements_list_for_intact_parts(text, placeholders_for_skipping_replacements)
    sorted_replacements_list_for_intact_parts = sorted(replacements_list_for_intact_parts, key=lambda x: len(x[0]), reverse=True)
    for original, place_holder_ in sorted_replacements_list_for_intact_parts:
        text = text.replace(original, place_holder_)

    # 4) @...@ स्थानीय प्रतिस्थापन
    tmp_replacements_list_for_localized_string_2 = create_replacements_list_for_localized_replacement(
        text, placeholders_for_localized_replacement, replacements_list_for_localized_string
    )
    sorted_replacements_list_for_localized_string = sorted(tmp_replacements_list_for_localized_string_2, key=lambda x: len(x[0]), reverse=True)
    for original, place_holder_, replaced_original in sorted_replacements_list_for_localized_string:
        text = text.replace(original, place_holder_)

    # 5) वैश्विक प्रतिस्थापन (old, new, placeholder)
    valid_replacements = {}
    for old, new, placeholder in replacements_final_list:
        if old in text:
            text = text.replace(old, placeholder)
            valid_replacements[placeholder] = new

    # 6) 2-अक्षर शब्दमूल प्रतिस्थापन (2 बार)
    valid_replacements_for_2char_roots = {}
    for old, new, placeholder in replacements_list_for_2char:
        if old in text:
            text = text.replace(old, placeholder)
            valid_replacements_for_2char_roots[placeholder] = new

    valid_replacements_for_2char_roots_2 = {}
    for old, new, placeholder in replacements_list_for_2char:
        if old in text:
            place_holder_second = "!" + placeholder + "!"
            text = text.replace(old, place_holder_second)
            valid_replacements_for_2char_roots_2[place_holder_second] = new

    # 7) प्लेसहोल्डर्स को अंतिम स्ट्रिंग्स में वापस बदलना
    for place_holder_second, new in reversed(valid_replacements_for_2char_roots_2.items()):
        text = text.replace(place_holder_second, new)
    for placeholder, new in reversed(valid_replacements_for_2char_roots.items()):
        text = text.replace(placeholder, new)
    for placeholder, new in valid_replacements.items():
        text = text.replace(placeholder, new)

    # स्थानीय(@)/स्किप(%) की पुनर्स्थापना
    for original, place_holder_, replaced_original in sorted_replacements_list_for_localized_string:
        text = text.replace(place_holder_, replaced_original.replace("@",""))
    for original, place_holder_ in sorted_replacements_list_for_intact_parts:
        text = text.replace(place_holder_, original.replace("%",""))

    # 8) HTML प्रारूप होने पर, न्यूलाइन को <br> में बदलना + स्पेस को &nbsp; में बदलना
    if "HTML" in format_type:
        text = text.replace("\n", "<br>\n")
        text = re.sub(r"   ", "&nbsp;&nbsp;&nbsp;", text)  # 3+ स्पेस को बदलना
        text = re.sub(r"  ", "&nbsp;&nbsp;", text)  # 2+ स्पेस को बदलना

    return text
```

यह एप्लिकेशन का सबसे महत्वपूर्ण फंक्शन है जो पूरी प्रतिस्थापन प्रक्रिया को संचालित करता है। यह तार्किक ढांग से संरचित है और कई चरणों में प्रोसेसिंग करता है:

1. **स्पेस मानकीकरण और अक्षर प्रारूप**: पहले स्पेस को मानकीकृत करें और एस्पेरांतो अक्षरों को सर्कमफ्लेक्स प्रारूप (ĉ, ĝ, आदि) में बदलें
2. **स्किप क्षेत्रों का संरक्षण**: %...% से घिरे पाठ को प्लेसहोल्डर्स से बदलें ताकि प्रतिस्थापन से बचाया जा सके
3. **स्थानीय प्रतिस्थापन**: @...@ से घिरे पाठ को अलग प्रतिस्थापन नियमों के साथ प्रोसेस करें
4. **वैश्विक प्रतिस्थापन**: मुख्य प्रतिस्थापन सूची लागू करें
5. **2-अक्षर शब्दमूल प्रतिस्थापन**: छोटे एस्पेरांतो शब्दमूलों के लिए विशेष प्रतिस्थापन (दो बार, ताकि ओवरलैपिंग केस हैंडल किए जा सकें)
6. **प्लेसहोल्डर्स की पुनर्स्थापना**: सभी प्लेसहोल्डर्स को उनके प्रोसेस्ड/अप्रोसेस्ड पाठ से बदलें
7. **HTML फॉर्मेटिंग**: यदि आवश्यक हो, तो HTML-विशिष्ट फॉर्मेटिंग लागू करें

### 3.6 समानांतर प्रोसेसिंग फंक्शंस

```python
def process_segment(
    lines: List[str],
    placeholders_for_skipping_replacements: List[str],
    replacements_list_for_localized_string: List[Tuple[str, str, str]],
    placeholders_for_localized_replacement: List[str],
    replacements_final_list: List[Tuple[str, str, str]],
    replacements_list_for_2char: List[Tuple[str, str, str]],
    format_type: str
) -> str:
    """
    मल्टीप्रोसेसिंग के लिए हेल्पर फंक्शन।
    lines (स्ट्रिंग लिस्ट) को जोड़ता है और फिर orchestrate_comprehensive_esperanto_text_replacement चलाता है।
    """
    segment = ''.join(lines)
    result = orchestrate_comprehensive_esperanto_text_replacement(
        segment,
        placeholders_for_skipping_replacements,
        replacements_list_for_localized_string,
        placeholders_for_localized_replacement,
        replacements_final_list,
        replacements_list_for_2char,
        format_type
    )
    return result

def parallel_process(
    text: str,
    num_processes: int,
    placeholders_for_skipping_replacements: List[str],
    replacements_list_for_localized_string: List[Tuple[str, str, str]],
    placeholders_for_localized_replacement: List[str],
    replacements_final_list: List[Tuple[str, str, str]],
    replacements_list_for_2char: List[Tuple[str, str, str]],
    format_type: str
) -> str:
    """
    दिए गए text को लाइन-बाई-लाइन विभाजित करता है, process_segment को
    मल्टीप्रोसेस में पैरेलल चलाता है और परिणाम जोड़ता है।
    """
    if num_processes <= 1:
        # सिंगल कोर के लिए सीधे orchestrate_comprehensive_esperanto_text_replacement कॉल करें
        return orchestrate_comprehensive_esperanto_text_replacement(
            text,
            placeholders_for_skipping_replacements,
            replacements_list_for_localized_string,
            placeholders_for_localized_replacement,
            replacements_final_list,
            replacements_list_for_2char,
            format_type
        )

    # लाइन्स के हिसाब से विभाजित करें (न्यूलाइन सहित)
    lines = re.findall(r'.*?\n|.+, text)
    num_lines = len(lines)

    if num_lines <= 1:
        # अगर एक या कम लाइनें हैं तो पैरेलल प्रोसेसिंग का कोई फायदा नहीं
        return orchestrate_comprehensive_esperanto_text_replacement(
            text,
            placeholders_for_skipping_replacements,
            replacements_list_for_localized_string,
            placeholders_for_localized_replacement,
            replacements_final_list,
            replacements_list_for_2char,
            format_type
        )

    # प्रत्येक प्रोसेस के लिए लाइनें आवंटित करें
    lines_per_process = max(num_lines // num_processes, 1)
    ranges = [(i * lines_per_process, (i + 1) * lines_per_process) for i in range(num_processes)]
    # अंतिम प्रोसेस को बाकी सब आवंटित करें
    ranges[-1] = (ranges[-1][0], num_lines)

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

    return ''.join(results)
```

ये फंक्शन्स ज़्यादा बड़े एस्पेरांतो पाठों के लिए समानांतर प्रोसेसिंग को लागू करते हैं:

- **process_segment**: स्ट्रिंग्स की एक सबसेट प्रोसेस करता है और `orchestrate_comprehensive_esperanto_text_replacement` कॉल करता है
- **parallel_process**: पूरे टेक्स्ट को लाइनों में विभाजित करता है, उन्हें प्रोसेसर्स के बीच वितरित करता है, और फिर परिणामों को जोड़ता है

समानांतर प्रोसेसिंग के लिए प्रमुख लॉजिक:
1. टेक्स्ट को लाइनों में विभाजित करें
2. प्रत्येक प्रोसेसर को लाइनों का एक सबसेट आवंटित करें
3. प्रत्येक सबसेट को समानांतर में प्रोसेस करें
4. परिणामों को फिर से जोड़ें

यह बड़े पाठों को प्रोसेस करते समय प्रदर्शन को महत्वपूर्ण रूप से बढ़ा सकता है।

### 3.7 HTML हेडर और फुटर

```python
def apply_ruby_html_header_and_footer(processed_text: str, format_type: str) -> str:
    """
    दिए गए आउटपुट प्रारूप के अनुसार, processed_text के लिए HTML हेडर और फुटर लागू करता है।
    उदाहरण: रूबी आकार समायोजन के लिए <style> टैग डालना।
    """
    if format_type in ('HTML格式_Ruby文字_大小调整','HTML格式_Ruby文字_大小调整_汉字替换'):
        # HTML प्रारूप में रूबी आकार बदलने के लिए स्टाइल
        ruby_style_head= """<!DOCTYPE html>
        <html lang="ja">
          <head>
            <meta charset="UTF-8">
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <title>大多数の环境中で正常に运行するRuby显示功能</title>
            <style>
            /* आकार समायोजन के लिए CSS स्टाइल... */
            </style>
          </head>
          <body>
          <p class="text-M_M">
        """
        ruby_style_tail = "</p></body></html>"
    elif format_type in ('HTML格式','HTML格式_汉字替换'):
        ruby_style_head = """<style>
        ruby rt {
            color: blue;
        }
        </style>
        """
        ruby_style_tail = "<br>"
    else:
        ruby_style_head = ""
        ruby_style_tail = ""

    return ruby_style_head + processed_text + ruby_style_tail
```

यह फंक्शन प्रोसेस्ड पाठ को उचित HTML हेडर और फुटर के साथ रैप करता है, जिसमें चुने गए प्रारूप के आधार पर CSS स्टाइलिंग शामिल है।

## 4. esp_replacement_json_make_module.py - प्रतिस्थापन JSON बनाने के लिए उपयोगिताएँ

`esp_replacement_json_make_module.py` में उपयोगिताएँ हैं जो उपयोगकर्ताओं को अपनी प्रतिस्थापन JSON फाइलें बनाने में मदद करती हैं।

### 4.1 मूलभूत अक्षर प्रतिस्थापन फंक्शन्स

```python
# ये फंक्शन्स esp_text_replacement_module.py में लगभग समान हैं
# (कोड दोहराव से बचने के लिए आगे लाइब्रेरी रिफैक्टरिंग में सुधार किया जा सकता है)

def replace_esperanto_chars(text, char_dict: Dict[str, str]) -> str:
    # ...

def convert_to_circumflex(text: str) -> str:
    # ...
```

### 4.2 पाठ चौड़ाई मापन और <br> इन्सर्शन

```python
def measure_text_width_Arial16(text, char_widths_dict: Dict[str, int]) -> int:
    """
    JSON से लोड किए गए {अक्षर: चौड़ाई(px)} शब्दकोश का उपयोग करके
    text की कुल चौड़ाई की गणना करता है
    """
    total_width = 0
    for ch in text:
        char_width = char_widths_dict.get(ch, 8)
        total_width += char_width
    return total_width

def insert_br_at_half_width(text, char_widths_dict: Dict[str, int]) -> str:
    """
    स्ट्रिंग की चौड़ाई आधी से अधिक होने पर <br> डालता है
    """
    # ...

def insert_br_at_third_width(text, char_widths_dict: Dict[str, int]) -> str:
    """
    स्ट्रिंग की चौड़ाई को तीन बराबर भागों में विभाजित करता है, और 1/3 और 2/3 स्थानों पर <br> डालता है
    """
    # ...
```

इन फंक्शंस का उपयोग रूबी एनोटेशन के लिए पाठ के लेआउट को नियंत्रित करने के लिए किया जाता है, खासकर जब मूल पाठ और रूबी के अनुपात असमान होते हैं:

- **measure_text_width_Arial16**: टेक्स्ट की चौड़ाई मापने के लिए प्रत्येक अक्षर की पिक्सेल चौड़ाई जोड़ता है
- **insert_br_at_half_width**: बहुत लंबे रूबी टेक्स्ट को आधे में विभाजित करने के लिए उपयोगी
- **insert_br_at_third_width**: अत्यधिक लंबे रूबी टेक्स्ट को तीन भागों में विभाजित करता है

### 4.3 आउटपुट प्रारूप

```python
def output_format(main_text, ruby_content, format_type, char_widths_dict):
    """
    एस्पेरांतो शब्दमूल (main_text) और उसके अनुवाद/काञ्जी (ruby_content) को
    निर्दिष्ट format_type में जोड़ता है
    """
    if format_type == 'HTML格式_Ruby文字_大小调整':
        width_ruby = measure_text_width_Arial16(ruby_content, char_widths_dict)
        width_main = measure_text_width_Arial16(main_text, char_widths_dict)
        ratio_1 = width_ruby / width_main
        if ratio_1 > 6:
            return f'<ruby>{main_text}<rt class="XXXS_S">{insert_br_at_third_width(ruby_content, char_widths_dict)}</rt></ruby>'
        elif ratio_1 > (9/3):
            return f'<ruby>{main_text}<rt class="XXS_S">{insert_br_at_half_width(ruby_content, char_widths_dict)}</rt></ruby>'
        # और कई अन्य अनुपात केस...

    elif format_type == 'HTML格式_Ruby文字_大小调整_汉字替换':
        # मुख्य और रूबी की भूमिकाओं को उल्टा करने जैसा प्रारूप
        # ...

    elif format_type == 'HTML格式':
        return f'<ruby>{main_text}<rt>{ruby_content}</rt></ruby>'

    elif format_type == 'HTML格式_汉字替换':
        return f'<ruby>{ruby_content}<rt>{main_text}</rt></ruby>'

    elif format_type == '括弧(号)格式':
        return f'{main_text}({ruby_content})'

    elif format_type == '括弧(号)格式_汉字替换':
        return f'{ruby_content}({main_text})'

    elif format_type == '替换后文字列のみ(仅)保留(简单替换)':
        return f'{ruby_content}'
```

यह फंक्शन विभिन्न आउटपुट प्रारूपों के लिए प्रतिस्थापन फॉर्मेटिंग लॉजिक का मुख्य केंद्र है:

1. **HTML रूबी (आकार समायोजन के साथ)**: मुख्य पाठ के ऊपर अनुवाद दिखाता है, चौड़ाई के अनुपात के आधार पर फॉन्ट साइज और लाइन ब्रेक्स समायोजित करता है
2. **HTML रूबी (आकार समायोजन और काञ्जी के साथ)**: उपरोक्त जैसा, लेकिन मुख्य पाठ और अनुवाद की भूमिकाएं उलट जाती हैं
3. **बेसिक HTML**: बेसिक HTML रूबी मार्कअप
4. **कोष्ठक प्रारूप**: अनुवाद को कोष्ठकों में दिखाता है
5. **केवल प्रतिस्थापित पाठ**: केवल अनुवादित अंश दिखाता है

### 4.4 मल्टीप्रोसेसिंग फंक्शन्स

```python
def process_chunk_for_pre_replacements(
    chunk: List[List[str]],
    replacements: List[Tuple[str, str, str]]
) -> Dict[str, List[str]]:
    """
    chunk: [[E_root, pos], ...] पार्शियल लिस्ट
    safe_replace द्वारा प्रतिस्थापन परिणाम { E_root: [replaced_stem, pos], ... } के रूप में रिटर्न करता है
    """
    # ...

def parallel_build_pre_replacements_dict(
    E_stem_with_Part_Of_Speech_list: List[List[str]],
    replacements: List[Tuple[str, str, str]],
    num_processes: int = 4
) -> Dict[str, List[str]]:
    """
    डेटा को num_processes हिस्सों में विभाजित करता है, process_chunk_for_pre_replacements को पैरेलल चलाता है
    और अंत में शब्दकोशों को मर्ज करके रिटर्न करता है।
    """
    # ...
```

ये फंक्शन्स प्रतिस्थापन शब्दकोश बनाने की प्रक्रिया को पैरेलल में संचालित करते हैं, विशेष रूप से बड़े डेटासेट के लिए प्रदर्शन में सुधार करते हैं।

### 4.5 समान रूबी निकालना

```python
IDENTICAL_RUBY_PATTERN = re.compile(r'<ruby>([^<]+)<rt class="XXL_L">([^<]+)</rt></ruby>')

def remove_redundant_ruby_if_identical(text: str) -> str:
    """
    <ruby>xxx<rt class="XXL_L">xxx</rt></ruby> के जैसे,
    अगर पैरेंट स्ट्रिंग और रूबी स्ट्रिंग पूरी तरह से समान हैं तो <ruby> को हटा देता है
    """
    # ...
```

यह फंक्शन अनावश्यक रूबी एनोटेशन को हटाकर आउटपुट को अनुकूलित करता है, जहां रूबी टेक्स्ट और मूल टेक्स्ट एक ही हैं।

## 5. एस्पेरान्तो पाठ को स्ट्रिंग (कांजी) से प्रतिस्थापित करने हेतु JSON फ़ाइल बनाने का पृष्ठ.py

इस फाइल में Streamlit इंटरफेस है जो उपयोगकर्ताओं को अपनी प्रतिस्थापन JSON फाइलें बनाने की सुविधा देता है।

### 5.1 मुख्य आयात और वैरिएबल

```python
import streamlit as st
import pandas as pd
import io
import os
import re
import json
import streamlit as st
from typing import List, Dict, Tuple, Optional
import multiprocessing
from io import StringIO
import streamlit.components.v1 as components

# दोनों यूटिलिटी मॉड्यूल से फंक्शन आयात
from esp_text_replacement_module import (
    convert_to_circumflex,
    safe_replace,
    import_placeholders,
    apply_ruby_html_header_and_footer
)
from esp_replacement_json_make_module import (
    convert_to_circumflex,
    output_format,
    import_placeholders,
    capitalize_ruby_and_rt,
    process_chunk_for_pre_replacements,
    parallel_build_pre_replacements_dict,
    remove_redundant_ruby_if_identical
)

# क्रिया प्रत्यय, AN, ON और अन्य डेटा को परिभाषित करना
verb_suffix_2l = {
    'as':'as', 'is':'is', 'os':'os', 'us':'us','at':'at','it':'it','ot':'ot',
    'ad':'ad','iĝ':'iĝ','ig':'ig','ant':'ant','int':'int','ont':'ont'
}

AN=[['dietan', '/diet/an/', '/diet/an'], ['afrikan', '/afrik/an/', '/afrik/an'],
   # और अधिक AN डेटा...
]
ON=[['duon', '/du/on/', '/du/on'], ['okon', '/ok/on/', '/ok/on'],
   # और अधिक ON डेटा...
]

# -1 जैसे वैल्यू के लिए अनुमति (यूजर द्वारा शब्द एक्सक्लूड करने के लिए)
allowed_values = {-1, "-1", "ー１", "ー1", "-１", "－１", "－1"}

# 2-अक्षर शब्दमूल प्रकार
suffix_2char_roots=['ad', 'ag', 'am', 'ar', 'as', 'at', 'av', 'di', 'ec', 'eg', 'ej', 'em', 'er', 'et', 'id', 'ig', 'il', 'in', 'ir', 'is', 'it', 'lu', 'nj', 'op', 'or', 'os', 'ot', 'ov', 'pi', 'te', 'uj', 'ul', 'um', 'us', 'uz','ĝu','aĵ','iĝ','aĉ','aĝ','ŝu','eĥ']
prefix_2char_roots=['al', 'am', 'av', 'bo', 'di', 'du', 'ek', 'el', 'en', 'fi', 'ge', 'ir', 'lu', 'ne', 'ok', 'or', 'ov', 'pi', 're', 'te', 'uz','ĝu','aĉ','aĝ','ŝu','eĥ']
standalone_2char_roots=['al', 'ci', 'da', 'de', 'di', 'do', 'du', 'el', 'en', 'fi', 'ha', 'he', 'ho', 'ia', 'ie', 'io', 'iu', 'ja', 'je', 'ju','ke', 'la', 'li', 'mi', 'ne', 'ni', 'nu', 'ok', 'ol', 'po', 'se', 'si', 've', 'vi','ŭa','aŭ','ĉe','ĝi','ŝi','ĉu']

# प्लेसहोल्डर फाइलें लोड करना
imported_placeholders_for_global_replacement = import_placeholders(
    './Appの运行に使用する各类文件/占位符(placeholders)_$20987$-$499999$_全域替换用.txt'
)
imported_placeholders_for_2char_replacement = import_placeholders(
    './Appの运行に使用する各类文件/占位符(placeholders)_$13246$-$19834$_二文字词根替换用.txt'
)
imported_placeholders_for_local_replacement = import_placeholders(
    './Appの运行に使用する各类文件/占位符(placeholders)_@20374@-@97648@_局部文字列替换用.txt'
)

# अक्षर चौड़ाई के लिए JSON लोड करना
with open("./Appの运行に使用する各类文件/Unicode_BMP全范围文字幅(宽)_Arial16.json", "r", encoding="utf-8") as fp:
    char_widths_dict = json.load(fp)
```

इस खंड में JSON जनरेटर पेज के लिए आवश्यक कई डेटा स्ट्रक्चर्स और वैरिएबल्स को सेट किया जाता है:

- **क्रिया प्रत्यय**: एस्पेरांतो क्रिया प्रत्ययों के लिए मैपिंग (as, is, os, आदि)
- **AN और ON लिस्ट्स**: विशेष सुफिक्स "an" और "on" वाले शब्दों के लिए डेटा
- **2-अक्षर शब्दमूल प्रकार**: एस्पेरांतो प्रीफिक्स, सुफिक्स, और स्टैंडअलोन 2-अक्षर शब्दमूलों की सूचियां
- **प्लेसहोल्डर्स**: विभिन्न प्रतिस्थापन प्रकारों के लिए प्लेसहोल्डर स्ट्रिंग्स
- **अक्षर चौड़ाई शब्दकोश**: यूनिकोड अक्षरों की वास्तविक रेंडर्ड चौड़ाई को स्टोर करता है

### 5.2 Streamlit इंटरफेस और CSV लोडिंग

```python
st.set_page_config(
    page_title="एस्पेरान्तो पाठ (चीनी अक्षर) प्रतिस्थापन के लिए JSON जनरेट टूल",
    layout="wide"
)
st.title("एस्पेरान्तो पाठ का (चीनी अक्षर) प्रतिस्थापन करने के लिए JSON फ़ाइल बनाएँ")
st.write("---")

# CSV फाइल चुनने के लिए यूजर इंटरफेस
st.header("चरण 1: CSV फ़ाइल की तैयारी")
st.markdown(
    """
### (एस्पेरान्तो शब्दमूल व चीनी अक्षर या हिंदी अनुवाद) वाली **CSV फ़ाइल** चुनें  
---
    """
)
csv_choice = st.radio("CSV फ़ाइल कैसे चुनें?", ("CSV अपलोड करें", "डिफ़ॉल्ट फ़ाइल प्रयोग करें"))
csv_path_default = "./Appの运行に使用する各类文件/एस्पेरान्तो शब्दमूलों की सूची हिंदी अनुवाद एवं रूबी टिप्पणियों सहित.csv"
CSV_data_imported = None

if csv_choice == "CSV अपलोड करें":
    # उपयोगकर्ता की CSV फाइल अपलोड करना और लोड करना
    uploaded_file = st.file_uploader("CSV फ़ाइल चुनें", type=['csv'])
    if uploaded_file is not None:
        file_contents = uploaded_file.read().decode("utf-8")
        converted_text = convert_to_circumflex(file_contents)
        csv_buffer = StringIO(converted_text)
        CSV_data_imported = pd.read_csv(csv_buffer, encoding="utf-8", usecols=[0, 1])
        st.success("CSV फ़ाइल सफलतापूर्वक अपलोड की गई!")
    else:
        st.warning("कोई CSV फ़ाइल अपलोड नहीं हुई।")
        st.stop()
elif csv_choice == "डिफ़ॉल्ट फ़ाइल प्रयोग करें":
    # डिफॉल्ट CSV फाइल लोड करना
    try:
        with open(csv_path_default, 'r', encoding="utf-8") as file:
            text = file.read()
        converted_text = convert_to_circumflex(text)
        csv_buffer = StringIO(converted_text)
        CSV_data_imported = pd.read_csv(csv_buffer, encoding="utf-8", usecols=[0, 1])
        st.info("डिफ़ॉल्ट CSV फ़ाइल का उपयोग किया जाएगा।")
    except FileNotFoundError:
        st.error("डिफ़ॉल्ट CSV फ़ाइल नहीं मिल सकी। प्रक्रिया रोकी जा रही है।")
        st.stop()
```

इस खंड में:
- **Streamlit पेज सेटअप**: JSON जनरेटर पेज के लिए टाइटल और लेआउट सेट करता है
- **CSV लोडिंग ऑप्शन्स**: उपयोगकर्ता या तो डिफॉल्ट CSV फाइल का उपयोग कर सकता है या अपनी खुद की अपलोड कर सकता है
- **CSV प्रोसेसिंग**: फाइल को `convert_to_circumflex` के साथ प्रोसेस करके एस्पेरांतो अक्षरों को मानकीकृत किया जाता है

CSV फाइल में एस्पेरांतो शब्दमूल और उनके अनुवाद जोड़े होते हैं, जो प्रतिस्थापन प्रक्रिया का आधार बनते हैं।

### 5.3 JSON सेटिंग्स फाइल्स लोड करना

```python
st.header("चरण 2: JSON फ़ाइल तैयार करें (शब्दमूल विखंडन इत्यादि)")
st.markdown("""
यहाँ आप **एस्पेरान्तो शब्दमूल विखंडन नियम** या **अपनी प्रतिस्थापन सेटिंग**  
(उपयोगकर्ता-परिभाषित) के JSON फ़ाइल को अपलोड कर सकते हैं, या डिफ़ॉल्ट फ़ाइल चुन सकते हैं।
""")

# 1) शब्दमूल विखंडन नियम वाली JSON
json_choice = st.radio("1) शब्दमूल विखंडन नियम वाली JSON:", ("अपलोड करें", "डिफ़ॉल्ट फ़ाइल"))
json_path_default = "./Appの运行に使用する各类文件/世界语单词词根分解方法の使用者自定义设置.json"
custom_stemming_setting_list = None

if json_choice == "अपलोड करें":
    # उपयोगकर्ता के जेएसओएन को लोड करना
    # ...
elif json_choice == "डिफ़ॉल्ट फ़ाइल":
    # डिफॉल्ट जेएसओएन को लोड करना
    # ...

# 2) प्रतिस्थापित होने वाले टेक्स्ट की JSON
json_choice2 = st.radio("2) प्रतिस्थापित होने वाले टेक्स्ट की JSON:", ("अपलोड करें", "डिफ़ॉल्ट फ़ाइल"))
json_path_default2 = "./Appの运行に使用する各类文件/替换后文字列(汉字)の使用者自定义设置(基本上完全不推荐).json"
user_replacement_item_setting_list = None

if json_choice2 == "अपलोड करें":
    # उपयोगकर्ता के जेएसओएन को लोड करना
    # ...
elif json_choice2 == "डिफ़ॉल्ट फ़ाइल":
    # डिफॉल्ट जेएसओएन को लोड करना
    # ...
```

यह खंड उपयोगकर्ता को दो महत्वपूर्ण JSON फाइलें लोड करने की अनुमति देता है:

1. **शब्दमूल विखंडन नियम**: इस JSON में एस्पेरांतो शब्दों को उनके शब्दमूलों में तोड़ने के नियम होते हैं
2. **प्रतिस्थापित टेक्स्ट सेटिंग्स**: यह JSON विशिष्ट शब्दों के लिए कस्टम प्रतिस्थापन नियम निर्दिष्ट करता है

ये सेटिंग्स उपयोगकर्ता को नियंत्रण प्रदान करती हैं कि प्रतिस्थापन प्रक्रिया कैसे काम करती है।

### 5.4 JSON जनरेशन प्रक्रिया

मुख्य प्रोसेसिंग को "प्रत्यस्थापन JSON फ़ाइल तैयार करें" बटन के माध्यम से शुरू किया जाता है:

```python
if st.button("प्रत्यस्थापन JSON फ़ाइल तैयार करें"):
    with st.spinner("JSON फ़ाइल जनरेट हो रही है, कृपया प्रतीक्षा करें..."):
        # 1) सभी एस्पेरांतो शब्दमूलों की सूची लोड करना
        with open("./Appの运行に使用する各类文件/PEJVO(世界语全部单词列表)'全部'について、词尾(a,i,u,e,o,n等)をcutし、comma(,)で隔てて词性と併せて记录した列表(E_stem_with_Part_Of_Speech_list).json", "r", encoding="utf-8") as g:
            E_stem_with_Part_Of_Speech_list = json.load(g)

        # 2) अस्थायी प्रतिस्थापन शब्दकोश बनाना
        temporary_replacements_dict = {}
        with open("./Appの运行に使用する各类文件/世界语全部词根_约11137个_202501.txt", 'r', encoding='utf-8') as file:
            E_roots = file.readlines()
            for E_root in E_roots:
                E_root = E_root.strip()
                if not E_root.isdigit():
                    temporary_replacements_dict[E_root] = [E_root, len(E_root)]

        # 3) CSV से शब्दमूल-अनुवाद जोड़े जोड़ना
        for *, (E*root, hanzi_or_meaning) in CSV_data_imported.iterrows():
            if pd.notna(E_root) and pd.notna(hanzi_or_meaning) \
               and '#' not in E_root and (E_root != '') and (hanzi_or_meaning != ''):
                temporary_replacements_dict[E_root] = [
                    output_format(E_root, hanzi_or_meaning, format_type, char_widths_dict),
                    len(E_root)
                ]

        # 4) सूची को लंबाई के आधार पर सॉर्ट करना (लंबे शब्द पहले)
        temporary_replacements_list_1 = []
        for old, new in temporary_replacements_dict.items():
            temporary_replacements_list_1.append((old, new[0], new[1]))
        temporary_replacements_list_2 = sorted(temporary_replacements_list_1, key=lambda x: x[2], reverse=True)

        # 5) प्लेसहोल्डर्स के साथ अंतिम सूची बनाना
        temporary_replacements_list_final = []
        for kk in range(len(temporary_replacements_list_2)):
            temporary_replacements_list_final.append([
                temporary_replacements_list_2[kk][0],
                temporary_replacements_list_2[kk][1],
                imported_placeholders_for_global_replacement[kk]
            ])

        # 6) पूर्व-प्रतिस्थापन शब्दकोश बनाना (पैरेलल या सिंगल-थ्रेड)
        if use_parallel:
            pre_replacements_dict_1 = parallel_build_pre_replacements_dict(
                E_stem_with_Part_Of_Speech_list,
                temporary_replacements_list_final,
                num_processes
            )
        else:
            # प्रोग्रेस दिखाने के साथ सिंगल-थ्रेड प्रोसेसिंग
            # ...

        # कुछ शब्दों को ड्रॉप करना (आवश्यकतानुसार)
        keys_to_remove = ['domen', 'teren','posten']
        for key in keys_to_remove:
            pre_replacements_dict_1.pop(key, None)

        # 7) pre_replacements_dict_1 को अधिक प्रोसेस करना
        # प्राथमिकता समायोजन, रूबी हटाना/रीसेट करना आदि
        pre_replacements_dict_2 = {}
        for i,j in pre_replacements_dict_1.items():
            # j[0] = safe_replace के बाद स्ट्रिंग, j[1] = संज्ञा प्रकार
            # i==j[0] - "वास्तव में प्रतिस्थापित नहीं शब्द" होने पर प्राथमिकता कम करें
            # ...

        # 8) AN, ON, क्रिया प्रत्यय आदि का उपयोग करके अधिकतर प्राथमिकता समायोजन
        # ...

        # 9) custom_stemming_setting_list (उपयोगकर्ता-परिभाषित शब्दमूल विखंडन) लागू करना
        # ...

        # 10) user_replacement_item_setting_list लागू करना (विशिष्ट शब्द-चीनी अक्षर प्रतिस्थापन)
        # ...

        # 11) pre_replacements_dict_3 से सूची बनाना और प्राथमिकता के आधार पर सॉर्ट करना
        # ...

        # 12) विभिन्न केस के लिए (अपरकेस/लोअरकेस/कैपिटलाइज़) वैरिएंट बनाना
        # ...

        # 13) 2-अक्षर शब्दमूल प्रतिस्थापन सूची बनाना
        # ...

        # 14) स्थानीय प्रतिस्थापन सूची बनाना (@...@ के लिए)
        # ...

        # 15) तीन प्रतिस्थापन सूचियों को जेएसओएन में मर्ज करना
        combined_data = {}
        combined_data["全域替换用のリスト(列表)型配列(replacements_final_list)"] = replacements_final_list
        combined_data["二文字词根替换用のリスト(列表)型配列(replacements_list_for_2char)"] = replacements_list_for_2char
        combined_data["局部文字替换用のリスト(列表)型配列(replacements_list_for_localized_string)"] = replacements_list_for_localized_string

        # 16) जेएसओएन डम्प और डाउनलोड बटन
        download_data = json.dumps(combined_data, ensure_ascii=False, indent=2)
        st.success("प्रतिस्थापन लिस्ट सफलतापूर्वक तैयार!")
        st.download_button(
            label="अंतिम प्रतिस्थापन लिस्ट (3 संयुक्त JSON) डाउनलोड",
            data=download_data,
            file_name="अंतिम_प्रतिस्थापन_लिस्ट(3_JSON).json",
            mime='application/json'
        )
```

इस जटिल प्रक्रिया में कई महत्वपूर्ण चरण शामिल हैं:

1. **एस्पेरांतो शब्दों का लोडिंग**: सभी एस्पेरांतो शब्दमूलों और उनके संज्ञा प्रकारों को लोड करना
2. **प्रारंभिक प्रतिस्थापन डिक्शनरी**: प्रत्येक शब्दमूल के लिए बेसिक प्रतिस्थापन बनाना
3. **CSV डेटा जोड़ना**: CSV से शब्दमूल-अनुवाद जोड़े जोड़ना
4. **लंबाई-आधारित सॉर्टिंग**: लंबे शब्दों को पहले प्रतिस्थापित करने के लिए सॉर्ट करना
5. **प्लेसहोल्डर असाइनमेंट**: प्रत्येक प्रतिस्थापन को एक अद्वितीय प्लेसहोल्डर असाइन करना
6. **संज्ञा-आधारित विस्तार**: संज्ञा प्रकार (संज्ञा, विशेषण, क्रिया) के आधार पर प्रतिस्थापन विस्तार करना
7. **विशेष प्रत्यय प्रोसेसिंग**: AN, ON, और क्रिया प्रत्ययों के लिए विशेष नियम लागू करना
8. **कस्टम नियम लागू करना**: उपयोगकर्ता-परिभाषित विखंडन और प्रतिस्थापन नियम लागू करना
9. **केस वैरिएंट**: लोअरकेस, अपरकेस, और कैपिटलाइज़्ड प्रतिस्थापन वैरिएंट बनाना
10. **फाइनल जेएसओएन**: तीन प्रमुख प्रतिस्थापन सूचियों को एक जेएसओएन फाइल में मर्ज करना

### 5.5 प्राथमिकताओं का निर्धारण

प्रतिस्थापन कोड का एक महत्वपूर्ण हिस्सा यह निर्धारित करना है कि विभिन्न प्रतिस्थापन की प्राथमिकता कैसे सेट की जाए:

```python
# संज्ञाओं के लिए उदाहरण प्राथमिकता असाइनमेंट
if "名词" in j[1]:  # 名词 = संज्ञा
    for k in ["o","on",'oj']:
        if not i+k in pre_replacements_dict_2:
            pre_replacements_dict_3[i+k]=[j[0]+k,j[2]+len(k)*10000-3000]
        elif j[0]+k != pre_replacements_dict_2[i+k][0]:
            pre_replacements_dict_3[i+k]=[j[0]+k,j[2]+len(k)*10000-3000]
            unchangeable_after_creation_list.append(i+k)
```

प्राथमिकता सिस्टम की मुख्य विशेषताएं:

- **लंबाई आधारित प्राथमिकता**: ज्यादातर मामलों में, लंबी स्ट्रिंग्स को उच्च प्राथमिकता दी जाती है, प्रतिस्थापन प्राथमिकता आमतौर पर `len(word) * 10000`
- **शब्दमूल प्रकार मॉडिफायर**: पूर्ण शब्दों (प्रत्यय के साथ) को आमतौर पर केवल शब्दमूलों की तुलना में अधिक प्राथमिकता दी जाती है
- **संज्ञा प्रकार सशर्त**: एक ही स्ट्रिंग संज्ञा, विशेषण, या क्रिया के आधार पर अलग-अलग तरह से प्रतिस्थापित हो सकती है
- **प्रत्यय मॉडिफायर्स**: "an", "on", क्रिया प्रत्यय, आदि को विशेष प्राथमिकता नियम दिए जाते हैं

इस जटिल प्राथमिकता सिस्टम का उद्देश्य यह सुनिश्चित करना है कि सबसे उपयुक्त प्रतिस्थापन चुने जाएं, विशेष रूप से जब विभिन्न प्रतिस्थापन नियम परस्पर विरोधी हो सकते हैं।

## 6. एप्लिकेशन कोड में उन्नत तकनीकें

इस एप्लिकेशन में कई उन्नत प्रोग्रामिंग तकनीकें लागू की गई हैं:

### 6.1 रिकर्सिव प्रतिस्थापन के लिए प्लेसहोल्डर पैटर्न

प्लेसहोल्डर पैटर्न का उपयोग एक महत्वपूर्ण समस्या को हल करता है:

```python
def safe_replace(text: str, replacements: List[Tuple[str, str, str]]) -> str:
    """
    (old, new, placeholder) की सूची प्राप्त करता है,
    text में old → placeholder → new के चरणबद्ध प्रतिस्थापन को करता है।
    """
    valid_replacements = {}# एस्पेरांतो पाठ प्रतिस्थापन एप्लिकेशन की तकनीकी संरचना का विश्लेषण

## परिचय

यह दस्तावेज़ एस्पेरांतो पाठ प्रतिस्थापन एप्लिकेशन के अंतर्निहित कोड संरचना और कार्यप्रणाली का विस्तृत विश्लेषण प्रदान करता है। एप्लिकेशन में चार मुख्य पायथन फाइलें शामिल हैं:

1. `main.py` - मुख्य एप्लिकेशन फाइल जो Streamlit इंटरफेस प्रदान करती है
2. `एस्पेरान्तो पाठ को स्ट्रिंग (कांजी) से प्रतिस्थापित करने हेतु JSON फ़ाइल बनाने का पृष्ठ.py` - JSON जनरेटर पेज
3. `esp_text_replacement_module.py` - टेक्स्ट प्रतिस्थापन यूटिलिटी मॉड्यूल
4. `esp_replacement_json_make_module.py` - JSON फाइल जनरेशन के लिए उपयोगिताएँ

इस दस्तावेज़ में हम प्रत्येक फाइल की संरचना, मुख्य फंक्शन, और उनके पारस्परिक संबंधों पर चर्चा करेंगे ताकि आप इस एप्लिकेशन के आर्किटेक्चर को गहराई से समझ सकें।

## 1. उच्च स्तरीय आर्किटेक्चर ओवरव्यू

### 1.1 एप्लिकेशन संरचना

एप्लिकेशन एक Streamlit वेब ऐप है जो मूलतः दो पेज प्रदान करता है:

- **मुख्य पेज** (`main.py`): एस्पेरांतो पाठ को काञ्जी (चीनी अक्षर) या अन्य भाषाओं के समकक्ष में परिवर्तित करता है, HTML रूबी एनोटेशन या अन्य प्रारूपों में।
- **JSON जनरेटर पेज** (`एस्पेरान्तो पाठ को...`): उपयोगकर्ताओं को प्रतिस्थापन नियमों वाली अपनी JSON फाइलें बनाने की अनुमति देता है।

### 1.2 डेटा प्रवाह

एप्लिकेशन में डेटा प्रवाह इस प्रकार है:

1. उपयोगकर्ता या तो डिफॉल्ट JSON प्रतिस्थापन फाइल का उपयोग करता है या अपनी खुद की अपलोड करता है
2. उपयोगकर्ता एस्पेरांतो पाठ इनपुट करता है (टाइप करके या फाइल अपलोड करके)
3. पाठ को एक टेक्स्ट प्रोसेसिंग पाइपलाइन (निम्न में से कई चरणों में) से गुजारा जाता है:
   - एस्पेरांतो विशेष अक्षरों को मानकीकृत करना
   - प्रतिस्थापन से बचाए जाने वाले क्षेत्रों को चिह्नित करना (प्लेसहोल्डर्स का उपयोग करके)
   - स्थानीय प्रतिस्थापन लागू करना
   - वैश्विक प्रतिस्थापन लागू करना
   - विशेष 2-अक्षर प्रतिस्थापन लागू करना
   - प्लेसहोल्डर्स की पुनर्स्थापना
4. परिवर्तित पाठ HTML या अन्य प्रारूपों में प्रदर्शित किया जाता है और डाउनलोड के लिए उपलब्ध कराया जाता है

### 1.3 मॉड्यूल के बीच संबंध

मॉड्यूल के बीच संबंध इस प्रकार हैं:

- `main.py` टेक्स्ट प्रतिस्थापन के लिए `esp_text_replacement_module.py` से फंक्शन आयात करता है
- `एस्पेरान्तो पाठ को...` फाइल दोनों `esp_text_replacement_module.py` और `esp_replacement_json_make_module.py` से फंक्शंस आयात करती है
- दोनों यूटिलिटी मॉड्यूल कुछ समान फंक्शन्स को परिभाषित करते हैं (कुछ दोहराव के साथ) लेकिन अलग-अलग उद्देश्यों के लिए

अब हम प्रत्येक फाइल के अंदर गहराई से जाएंगे और उनकी प्रमुख विशेषताओं पर चर्चा करेंगे।

## 2. main.py - मुख्य एप्लिकेशन फाइल

`main.py` एप्लिकेशन की मुख्य फाइल है जो ग्राफिकल इंटरफेस और टेक्स्ट प्रतिस्थापन की मुख्य कार्यक्षमता प्रदान करती है।

### 2.1 आयात और सेटअप

```python
import streamlit as st
import re
import io
import json
import pandas as pd  # यदि आवश्यक हो
from typing import List, Dict, Tuple, Optional
import streamlit.components.v1 as components
import multiprocessing

# multiprocessing के लिए 'spawn' मोड सेट करना
try:
    multiprocessing.set_start_method("spawn")
except RuntimeError:
    pass  # यदि विधि पहले से सेट है तो इसे अनदेखा करें

# esp_text_replacement_module से फंक्शन आयात
from esp_text_replacement_module import (
    x_to_circumflex,
    x_to_hat,
    hat_to_circumflex,
    circumflex_to_hat,
    replace_esperanto_chars,
    import_placeholders,
    orchestrate_comprehensive_esperanto_text_replacement,
    parallel_process,
    apply_ruby_html_header_and_footer
)
```

महत्वपूर्ण बिंदु:
- **multiprocessing**: एप्लिकेशन समानांतर प्रोसेसिंग के लिए पायथन के `multiprocessing` मॉड्यूल का उपयोग करता है (`spawn` मोड में)
- **टाइप हिंट्स**: कोड टाइप हिंट्स (`List`, `Dict`, आदि) का उपयोग करता है जो पायथन टाइप सिस्टम का सुझाव देता है
- **esp_text_replacement_module**: एप्लिकेशन एस्पेरांतो अक्षरों और टेक्स्ट प्रतिस्थापन से संबंधित विभिन्न फंक्शन आयात करता है

### 2.2 JSON फाइल लोडिंग

```python
@st.cache_data
def load_replacements_lists(json_path: str) -> Tuple[List, List, List]:
    """
    JSON फाइल लोड करता है और 3 लिस्ट्स का टपल रिटर्न करता है:
    1) replacements_final_list
    2) replacements_list_for_localized_string
    3) replacements_list_for_2char
    """
    with open(json_path, 'r', encoding='utf-8') as f:
        data = json.load(f)
    replacements_final_list = data.get(
        "全域替换用のリスト(列表)型配列(replacements_final_list)", []
    )
    replacements_list_for_localized_string = data.get(
        "局部文字替换用のリスト(列表)型配列(replacements_list_for_localized_string)", []
    )
    replacements_list_for_2char = data.get(
        "二文字词根替换用のリスト(列表)型配列(replacements_list_for_2char)", []
    )
    return (
        replacements_final_list,
        replacements_list_for_localized_string,
        replacements_list_for_2char,
    )
```

महत्वपूर्ण बिंदु:
- **@st.cache_data**: यह Streamlit डेकोरेटर फंक्शन के परिणामों को कैश करता है, जिससे बड़ी JSON फाइलों को बार-बार लोड करने से बचा जा सकता है
- **तीन प्रकार के प्रतिस्थापन**: फंक्शन तीन अलग प्रतिस्थापन सूचियां रिटर्न करता है, प्रत्येक का अपना विशिष्ट उद्देश्य है:
  1. `replacements_final_list` - वैश्विक प्रतिस्थापन के लिए
  2. `replacements_list_for_localized_string` - स्थानीय प्रतिस्थापन के लिए (@...@ के अंदर)
  3. `replacements_list_for_2char` - विशेष रूप से 2-अक्षर वाले एस्पेरांतो रूट्स के लिए

### 2.3 Streamlit पेज सेटअप

```python
st.set_page_config(
    page_title="एस्पेरांतो पाठ में कैरेक्टर (कान्जी) बदलने का उपकरण",
    layout="wide"
)
# शीर्षक (हिंदी में GUI प्रदर्शन के लिए)
st.title("एस्पेरांतो पाठ को कान्जी या HTML एनोटेशन से बदलना (विस्तारित संस्करण)")
st.write("---")
```

यह खंड Streamlit पेज के लेआउट और शीर्षक को कॉन्फ़िगर करता है।

### 2.4 JSON फाइल का लोडिंग और प्लेसहोल्डर का आयात

```python
# JSON फाइल लोड करना
json_options = ["デフォルトを使用する", "アップロードする"]
selected_option = st.radio(
    "JSON फ़ाइल के साथ कैसे कार्य करना है? (रिप्लेसमेंट JSON फ़ाइल को लोड करने के लिए)",
    json_options,
    format_func=lambda x: "डिफ़ॉल्ट JSON का उपयोग करें" if x == "デフォルトを使用する" else "फ़ाइल अपलोड करें"
)

# प्लेसहोल्डर्स लोड करना
placeholders_for_skipping_replacements: List[str] = import_placeholders(
    './Appの运行に使用する各类文件/占位符(placeholders)_%1854%-%4934%_文字列替换skip用.txt'
)
placeholders_for_localized_replacement: List[str] = import_placeholders(
    './Appの运行に使用する各类文件/占位符(placeholders)_@5134@-@9728@_局部文字列替换结果捕捉用.txt'
)
```

महत्वपूर्ण बिंदु:
- **JSON विकल्प**: उपयोगकर्ता या तो डिफॉल्ट JSON फाइल का उपयोग कर सकता है या अपनी खुद की अपलोड कर सकता है
- **प्लेसहोल्डर्स**: दो प्लेसहोल्डर फाइलें लोड की जाती हैं:
  1. `placeholders_for_skipping_replacements` - प्रतिस्थापन से बचे हुए पाठ (%...%) को अस्थायी रूप से स्टोर करने के लिए
  2. `placeholders_for_localized_replacement` - स्थानीय प्रतिस्थापन (@...@) के लिए

### 2.5 टेक्स्ट प्रोसेसिंग फॉर्म और प्रतिस्थापन कार्यान्वयन

```python
with st.form(key='profile_form'):
    # टेक्स्ट इनपुट, लेटर प्रकार चयन, आदि...

    submit_btn = st.form_submit_button('सबमिट')
    cancel_btn = st.form_submit_button("रद्द करें")

    if cancel_btn:
        st.warning("कार्रवाई रद्द कर दी गई।")
        st.stop()

    if submit_btn:
        st.session_state["text0_value"] = text0
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
                text=text0,
                placeholders_for_skipping_replacements=placeholders_for_skipping_replacements,
                replacements_list_for_localized_string=replacements_list_for_localized_string,
                placeholders_for_localized_replacement=placeholders_for_localized_replacement,
                replacements_final_list=replacements_final_list,
                replacements_list_for_2char=replacements_list_for_2char,
                format_type=format_type
            )

        # विशेष अक्षर प्रतिस्थापन (x_to_circumflex, आदि) लागू करें
        if letter_type == '上付き文字':
            processed_text = replace_esperanto_chars(processed_text, x_to_circumflex)
            processed_text = replace_esperanto_chars(processed_text, hat_to_circumflex)
        elif letter_type == '^形式':
            processed_text = replace_esperanto_chars(processed_text, x_to_hat)
            processed_text = replace_esperanto_chars(processed_text, circumflex_to_hat)

        processed_text = apply_ruby_html_header_and_footer(processed_text, format_type)
```

महत्वपूर्ण बिंदु:
- **समानांतर/एकल थ्रेड प्रोसेसिंग**: उपयोगकर्ता के चयन के आधार पर, एप्लिकेशन या तो `parallel_process` या `orchestrate_comprehensive_esperanto_text_replacement` का उपयोग करता है
- **एस्पेरांतो अक्षर प्रतिस्थापन**: उपयोगकर्ता द्वारा चुने गए दिखावट के अनुसार एस्पेरांतो विशेष अक्षरों का प्रतिस्थापन किया जाता है
- **HTML हेडर/फुटर अप्लाई करना**: आउटपुट प्रारूप के आधार पर उपयुक्त HTML हेडर और फुटर जोड़े जाते हैं

### 2.6 परिणाम प्रदर्शन और डाउनलोड

```python
if processed_text:
    MAX_PREVIEW_LINES = 250
    lines = processed_text.splitlines()
    if len(lines) > MAX_PREVIEW_LINES:
        # बहुत बड़े पाठ के लिए आंशिक प्रीव्यू...
    else:
        preview_text = processed_text

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

    download_data = processed_text.encode('utf-8')
    st.download_button(
        label="परिणाम डाउनलोड करें",
        data=download_data,
        file_name="प्रतिस्थापन_परिणाम.html",
        mime="text/html"
    )
```

महत्वपूर्ण बिंदु:
- **बड़े पाठ के लिए आंशिक प्रीव्यू**: अत्यधिक लंबे पाठ के लिए, एप्लिकेशन केवल पहली 247 और अंतिम 3 लाइनें दिखाता है
- **HTML प्रीव्यू**: HTML प्रारूप के लिए, एप्लिकेशन दो टैब प्रदान करता है—एक रेंडर्ड HTML के लिए और एक सोर्स कोड के लिए
- **डाउनलोड बटन**: उपयोगकर्ता प्रोसेस्ड पाठ को HTML फाइल के रूप में डाउनलोड कर सकता है

## 3. esp_text_replacement_module.py - टेक्स्ट प्रतिस्थापन की मुख्य संरचना

`esp_text_replacement_module.py` एस्पेरांतो पाठ प्रतिस्थापन और प्रसंस्करण के लिए मुख्य कार्यात्मकता प्रदान करता है।

### 3.1 एस्पेरांतो अक्षर मैपिंग

```python
# ================================
# 1) एस्पेरांतो अक्षर रूपांतरण के लिए शब्दकोष
# ================================
x_to_circumflex = {
    'cx': 'ĉ', 'gx': 'ĝ', 'hx': 'ĥ', 'jx': 'ĵ', 'sx': 'ŝ', 'ux': 'ŭ',
    'Cx': 'Ĉ', 'Gx': 'Ĝ', 'Hx': 'Ĥ', 'Jx': 'Ĵ', 'Sx': 'Ŝ', 'Ux': 'Ŭ'
}
circumflex_to_x = {...}  # उलटा मैपिंग
x_to_hat = {...}  # 'cx' -> 'c^', आदि
hat_to_x = {...}  # 'c^' -> 'cx', आदि
hat_to_circumflex = {...}  # 'c^' -> 'ĉ', आदि
circumflex_to_hat = {...}  # 'ĉ' -> 'c^', आदि
```

यह खंड एस्पेरांतो विशेष अक्षरों के विभिन्न प्रतिनिधित्वों के बीच मैपिंग परिभाषित करता है:
- **x-सिस्टम**: 'cx', 'gx', आदि (आसान टाइपिंग के लिए)
- **हैट (^) सिस्टम**: 'c^', 'g^', आदि
- **सर्कमफ्लेक्स सिस्टम**: 'ĉ', 'ĝ', आदि (एस्पेरांतो के मानक अक्षर)

### 3.2 मूलभूत अक्षर प्रतिस्थापन फंक्शन्स

```python
def replace_esperanto_chars(text, char_dict: Dict[str, str]) -> str:
    # char_dict में निहित जोड़े (original_char, converted_char) के लिए
    # text.replace() करें
    for original_char, converted_char in char_dict.items():
        text = text.replace(original_char, converted_char)
    return text

def convert_to_circumflex(text: str) -> str:
    """
    पाठ को सर्कमफ्लेक्स प्रारूप (ĉ, ĝ, ĥ, ĵ, ŝ, ŭ आदि) में एकीकृत करता है।
    1. hat_to_circumflex: c^ → ĉ
    2. x_to_circumflex: cx → ĉ
    """
    text = replace_esperanto_chars(text, hat_to_circumflex)
    text = replace_esperanto_chars(text, x_to_circumflex)
    return text

def unify_halfwidth_spaces(text: str) -> str:
    """
    फुलविड्थ स्पेस (U+3000) को अपरिवर्तित छोड़ते हुए, हाफविड्थ स्पेस और विज़ुअली मिलते-जुलते
    स्पेस कैरेक्टर्स को ASCII हाफविड्थ स्पेस (U+0020) में एकीकृत करता है।
    """
    pattern = r"[\u00A0\u2002\u2003\u2004\u2005\u2006\u2007\u2008\u2009\u200A]"
    return re.sub(pattern, " ", text)
```

ये फंक्शन एस्पेरांतो अक्षरों के बीच रूपांतरण प्रदान करते हैं और स्पेस कैरेक्टर्स को एकीकृत करते हैं।

### 3.3 प्लेसहोल्डर और सुरक्षित प्रतिस्थापन

```python
def safe_replace(text: str, replacements: List[Tuple[str, str, str]]) -> str:
    """
    (old, new, placeholder) की सूची प्राप्त करता है,
    text में old → placeholder → new के चरणबद्ध प्रतिस्थापन को करता है।
    """
    valid_replacements = {}
    # पहले old→placeholder
    for old, new, placeholder in replacements:
        if old in text:
            text = text.replace(old, placeholder)
            valid_replacements[placeholder] = new
    # फिर placeholder→new
    for placeholder, new in valid_replacements.items():
        text = text.replace(placeholder, new)
    return text

def import_placeholders(filename: str) -> List[str]:
    """
    प्लेसहोल्डर्स को रेखा-दर-रेखा पढ़ने वाला सरल फंक्शन
    """
    with open(filename, 'r') as file:
        placeholders = [line.strip() for line in file if line.strip()]
    return placeholders
```

इन फंक्शन्स का उपयोग प्रतिस्थापन प्रक्रिया में किया जाता है:
- **safe_replace**: यह फंक्शन एक सुरक्षित प्रतिस्थापन तंत्र का उपयोग करता है जो पहले मूल टेक्स्ट को प्लेसहोल्डर से बदलता है और फिर प्लेसहोल्डर को अंतिम वैल्यू से बदलता है, इस प्रकार रिकर्सिव प्रतिस्थापन से बचता है
- **import_placeholders**: यह फंक्शन फाइल से प्लेसहोल्डर स्ट्रिंग्स को लाइन-बाय-लाइन पढ़ता है

### 3.4 विशेष पैटर्न की पहचान और प्रतिस्थापन सूचियाँ बनाना

```python
# '%' से घिरे क्षेत्रों को प्रतिस्थापन से छोड़ने के लिए रेगेक्स
PERCENT_PATTERN = re.compile(r'%(.{1,50}?)%')

def find_percent_enclosed_strings_for_skipping_replacement(text: str) -> List[str]:
    """'%foo%' के सभी प्रारूपों को निकालता है। 50 अक्षरों तक सीमित।"""
    # ...

def create_replacements_list_for_intact_parts(text: str, placeholders: List[str]) -> List[Tuple[str, str]]:
    """
    '%xxx%' से घिरे क्षेत्रों का पता लगाता है,
    ( '%xxx%', placeholder ) के प्रारूप में प्रत्येक को मैप करने वाली सूची बनाता है
    """
    # ...

# '@' से घिरे क्षेत्रों को स्थानीय प्रतिस्थापन के लिए रेगेक्स
AT_PATTERN = re.compile(r'@(.{1,18}?)@')

def find_at_enclosed_strings_for_localized_replacement(text: str) -> List[str]:
    """'@foo@' के सभी प्रारूपों को निकालता है। 18 अक्षरों तक सीमित।"""
    # ...

def create_replacements_list_for_localized_replacement(text, placeholders: List[str],
                                                       replacements_list_for_localized_string: List[Tuple[str, str, str]]
                                                       ) -> List[List[str]]:
    """
    '@xxx@' से घिरे क्षेत्रों का पता लगाता है,
    उनके अंदर के पाठ 'xxx' को replacements_list_for_localized_string के साथ बदलता है
    और परिणाम को प्लेसहोल्डर से बदलता है।
    """
    matches = find_at_enclosed_strings_for_localized_replacement(text)
    tmp_list = []
    for i, match in enumerate(matches):
        if i < len(placeholders):
            replaced_match = safe_replace(match, replacements_list_for_localized_string)
            tmp_list.append([f"@{match}@", placeholders[i], replaced_match])
        else:
            break
    return tmp_list

### 3.5 मुख्य प्रतिस्थापन ऑर्केस्ट्रेशन फंक्शन

```python
def orchestrate_comprehensive_esperanto_text_replacement(
    text,
    placeholders_for_skipping_replacements: List[str],
    replacements_list_for_localized_string: List[Tuple[str, str, str]],
    placeholders_for_localized_replacement: List[str],
    replacements_final_list: List[Tuple[str, str, str]],
    replacements_list_for_2char: List[Tuple[str, str, str]],
    format_type: str
) -> str:
    """
    एस्पेरांतो पाठ को विभिन्न रूपांतरण नियमों के अनुसार व्यापक रूप से प्रतिस्थापित करने वाला मुख्य फंक्शन।
    1) स्पेस का मानकीकरण → 2) एस्पेरांतो अक्षरों (ĉ आदि) को सर्कमफ्लेक्स प्रारूप में एकीकृत करना
    3) % से घिरे हिस्सों को छोड़ना
    4) @ से घिरे हिस्सों को स्थानीय रूप से प्रतिस्थापित करना
    5) वैश्विक प्रतिस्थापन
    6) 2-अक्षर शब्दमूलों को 2 बार प्रतिस्थापित करना
    7) प्लेसहोल्डर पुनर्स्थापना
    8) HTML प्रारूप निर्दिष्ट होने पर अतिरिक्त फॉर्मेटिंग
    """

    # 1, 2) स्पेस का मानकीकरण + एस्पेरांतो अक्षरों को सर्कमफ्लेक्स में परिवर्तन
    text = unify_halfwidth_spaces(text)
    text = convert_to_circumflex(text)

    # 3) %...% स्किप क्षेत्रों को अस्थायी रूप से प्रतिस्थापित करना
    replacements_list_for_intact_parts = create_replacements_list_for_intact_parts(text, placeholders_for_skipping_replacements)
    sorted_replacements_list_for_intact_parts = sorted(replacements_list_for_intact_parts, key=lambda x: len(x[0]), reverse=True)
    for original, place_holder_ in sorted_replacements_list_for_intact_parts:
        text = text.replace(original, place_holder_)

    # 4) @...@ स्थानीय प्रतिस्थापन
    tmp_replacements_list_for_localized_string_2 = create_replacements_list_for_localized_replacement(
        text, placeholders_for_localized_replacement, replacements_list_for_localized_string
    )
    sorted_replacements_list_for_localized_string = sorted(tmp_replacements_list_for_localized_string_2, key=lambda x: len(x[0]), reverse=True)
    for original, place_holder_, replaced_original in sorted_replacements_list_for_localized_string:
        text = text.replace(original, place_holder_)

    # 5) वैश्विक प्रतिस्थापन (old, new, placeholder)
    valid_replacements = {}
    for old, new, placeholder in replacements_final_list:
        if old in text:
            text = text.replace(old, placeholder)
            valid_replacements[placeholder] = new

    # 6) 2-अक्षर शब्दमूल प्रतिस्थापन (2 बार)
    valid_replacements_for_2char_roots = {}
    for old, new, placeholder in replacements_list_for_2char:
        if old in text:
            text = text.replace(old, placeholder)
            valid_replacements_for_2char_roots[placeholder] = new

    valid_replacements_for_2char_roots_2 = {}
    for old, new, placeholder in replacements_list_for_2char:
        if old in text:
            place_holder_second = "!" + placeholder + "!"
            text = text.replace(old, place_holder_second)
            valid_replacements_for_2char_roots_2[place_holder_second] = new

    # 7) प्लेसहोल्डर्स को अंतिम स्ट्रिंग्स में वापस बदलना
    for place_holder_second, new in reversed(valid_replacements_for_2char_roots_2.items()):
        text = text.replace(place_holder_second, new)
    for placeholder, new in reversed(valid_replacements_for_2char_roots.items()):
        text = text.replace(placeholder, new)
    for placeholder, new in valid_replacements.items():
        text = text.replace(placeholder, new)

    # स्थानीय(@)/स्किप(%) की पुनर्स्थापना
    for original, place_holder_, replaced_original in sorted_replacements_list_for_localized_string:
        text = text.replace(place_holder_, replaced_original.replace("@",""))
    for original, place_holder_ in sorted_replacements_list_for_intact_parts:
        text = text.replace(place_holder_, original.replace("%",""))

    # 8) HTML प्रारूप होने पर, न्यूलाइन को <br> में बदलना + स्पेस को &nbsp; में बदलना
    if "HTML" in format_type:
        text = text.replace("\n", "<br>\n")
        text = re.sub(r"   ", "&nbsp;&nbsp;&nbsp;", text)  # 3+ स्पेस को बदलना
        text = re.sub(r"  ", "&nbsp;&nbsp;", text)  # 2+ स्पेस को बदलना

    return text
```

यह एप्लिकेशन का सबसे महत्वपूर्ण फंक्शन है जो पूरी प्रतिस्थापन प्रक्रिया को संचालित करता है। यह तार्किक ढांग से संरचित है और कई चरणों में प्रोसेसिंग करता है:

1. **स्पेस मानकीकरण और अक्षर प्रारूप**: पहले स्पेस को मानकीकृत करें और एस्पेरांतो अक्षरों को सर्कमफ्लेक्स प्रारूप (ĉ, ĝ, आदि) में बदलें
2. **स्किप क्षेत्रों का संरक्षण**: %...% से घिरे पाठ को प्लेसहोल्डर्स से बदलें ताकि प्रतिस्थापन से बचाया जा सके
3. **स्थानीय प्रतिस्थापन**: @...@ से घिरे पाठ को अलग प्रतिस्थापन नियमों के साथ प्रोसेस करें
4. **वैश्विक प्रतिस्थापन**: मुख्य प्रतिस्थापन सूची लागू करें
5. **2-अक्षर शब्दमूल प्रतिस्थापन**: छोटे एस्पेरांतो शब्दमूलों के लिए विशेष प्रतिस्थापन (दो बार, ताकि ओवरलैपिंग केस हैंडल किए जा सकें)
6. **प्लेसहोल्डर्स की पुनर्स्थापना**: सभी प्लेसहोल्डर्स को उनके प्रोसेस्ड/अप्रोसेस्ड पाठ से बदलें
7. **HTML फॉर्मेटिंग**: यदि आवश्यक हो, तो HTML-विशिष्ट फॉर्मेटिंग लागू करें

### 3.6 समानांतर प्रोसेसिंग फंक्शंस

```python
def process_segment(
    lines: List[str],
    placeholders_for_skipping_replacements: List[str],
    replacements_list_for_localized_string: List[Tuple[str, str, str]],
    placeholders_for_localized_replacement: List[str],
    replacements_final_list: List[Tuple[str, str, str]],
    replacements_list_for_2char: List[Tuple[str, str, str]],
    format_type: str
) -> str:
    """
    मल्टीप्रोसेसिंग के लिए हेल्पर फंक्शन।
    lines (स्ट्रिंग लिस्ट) को जोड़ता है और फिर orchestrate_comprehensive_esperanto_text_replacement चलाता है।
    """
    segment = ''.join(lines)
    result = orchestrate_comprehensive_esperanto_text_replacement(
        segment,
        placeholders_for_skipping_replacements,
        replacements_list_for_localized_string,
        placeholders_for_localized_replacement,
        replacements_final_list,
        replacements_list_for_2char,
        format_type
    )
    return result

def parallel_process(
    text: str,
    num_processes: int,
    placeholders_for_skipping_replacements: List[str],
    replacements_list_for_localized_string: List[Tuple[str, str, str]],
    placeholders_for_localized_replacement: List[str],
    replacements_final_list: List[Tuple[str, str, str]],
    replacements_list_for_2char: List[Tuple[str, str, str]],
    format_type: str
) -> str:
    """
    दिए गए text को लाइन-बाई-लाइन विभाजित करता है, process_segment को
    मल्टीप्रोसेस में पैरेलल चलाता है और परिणाम जोड़ता है।
    """
    if num_processes <= 1:
        # सिंगल कोर के लिए सीधे orchestrate_comprehensive_esperanto_text_replacement कॉल करें
        return orchestrate_comprehensive_esperanto_text_replacement(
            text,
            placeholders_for_skipping_replacements,
            replacements_list_for_localized_string,
            placeholders_for_localized_replacement,
            replacements_final_list,
            replacements_list_for_2char,
            format_type
        )

    # लाइन्स के हिसाब से विभाजित करें (न्यूलाइन सहित)
    lines = re.findall(r'.*?\n|.+, text)
    num_lines = len(lines)

    if num_lines <= 1:
        # अगर एक या कम लाइनें हैं तो पैरेलल प्रोसेसिंग का कोई फायदा नहीं
        return orchestrate_comprehensive_esperanto_text_replacement(
            text,
            placeholders_for_skipping_replacements,
            replacements_list_for_localized_string,
            placeholders_for_localized_replacement,
            replacements_final_list,
            replacements_list_for_2char,
            format_type
        )

    # प्रत्येक प्रोसेस के लिए लाइनें आवंटित करें
    lines_per_process = max(num_lines // num_processes, 1)
    ranges = [(i * lines_per_process, (i + 1) * lines_per_process) for i in range(num_processes)]
    # अंतिम प्रोसेस को बाकी सब आवंटित करें
    ranges[-1] = (ranges[-1][0], num_lines)

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

    return ''.join(results)
```

ये फंक्शन्स ज़्यादा बड़े एस्पेरांतो पाठों के लिए समानांतर प्रोसेसिंग को लागू करते हैं:

- **process_segment**: स्ट्रिंग्स की एक सबसेट प्रोसेस करता है और `orchestrate_comprehensive_esperanto_text_replacement` कॉल करता है
- **parallel_process**: पूरे टेक्स्ट को लाइनों में विभाजित करता है, उन्हें प्रोसेसर्स के बीच वितरित करता है, और फिर परिणामों को जोड़ता है

समानांतर प्रोसेसिंग के लिए प्रमुख लॉजिक:
1. टेक्स्ट को लाइनों में विभाजित करें
2. प्रत्येक प्रोसेसर को लाइनों का एक सबसेट आवंटित करें
3. प्रत्येक सबसेट को समानांतर में प्रोसेस करें
4. परिणामों को फिर से जोड़ें

यह बड़े पाठों को प्रोसेस करते समय प्रदर्शन को महत्वपूर्ण रूप से बढ़ा सकता है।

### 3.7 HTML हेडर और फुटर

```python
def apply_ruby_html_header_and_footer(processed_text: str, format_type: str) -> str:
    """
    दिए गए आउटपुट प्रारूप के अनुसार, processed_text के लिए HTML हेडर और फुटर लागू करता है।
    उदाहरण: रूबी आकार समायोजन के लिए <style> टैग डालना।
    """
    if format_type in ('HTML格式_Ruby文字_大小调整','HTML格式_Ruby文字_大小调整_汉字替换'):
        # HTML प्रारूप में रूबी आकार बदलने के लिए स्टाइल
        ruby_style_head= """<!DOCTYPE html>
        <html lang="ja">
          <head>
            <meta charset="UTF-8">
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <title>大多数の环境中で正常に运行するRuby显示功能</title>
            <style>
            /* आकार समायोजन के लिए CSS स्टाइल... */
            </style>
          </head>
          <body>
          <p class="text-M_M">
        """
        ruby_style_tail = "</p></body></html>"
    elif format_type in ('HTML格式','HTML格式_汉字替换'):
        ruby_style_head = """<style>
        ruby rt {
            color: blue;
        }
        </style>
        """
        ruby_style_tail = "<br>"
    else:
        ruby_style_head = ""
        ruby_style_tail = ""

    return ruby_style_head + processed_text + ruby_style_tail
```

यह फंक्शन प्रोसेस्ड पाठ को उचित HTML हेडर और फुटर के साथ रैप करता है, जिसमें चुने गए प्रारूप के आधार पर CSS स्टाइलिंग शामिल है।

## 4. esp_replacement_json_make_module.py - प्रतिस्थापन JSON बनाने के लिए उपयोगिताएँ

`esp_replacement_json_make_module.py` में उपयोगिताएँ हैं जो उपयोगकर्ताओं को अपनी प्रतिस्थापन JSON फाइलें बनाने में मदद करती हैं।

### 4.1 मूलभूत अक्षर प्रतिस्थापन फंक्शन्स

```python
# ये फंक्शन्स esp_text_replacement_module.py में लगभग समान हैं
# (कोड दोहराव से बचने के लिए आगे लाइब्रेरी रिफैक्टरिंग में सुधार किया जा सकता है)

def replace_esperanto_chars(text, char_dict: Dict[str, str]) -> str:
    # ...

def convert_to_circumflex(text: str) -> str:
    # ...
```

### 4.2 पाठ चौड़ाई मापन और <br> इन्सर्शन

```python
def measure_text_width_Arial16(text, char_widths_dict: Dict[str, int]) -> int:
    """
    JSON से लोड किए गए {अक्षर: चौड़ाई(px)} शब्दकोश का उपयोग करके
    text की कुल चौड़ाई की गणना करता है
    """
    total_width = 0
    for ch in text:
        char_width = char_widths_dict.get(ch, 8)
        total_width += char_width
    return total_width

def insert_br_at_half_width(text, char_widths_dict: Dict[str, int]) -> str:
    """
    स्ट्रिंग की चौड़ाई आधी से अधिक होने पर <br> डालता है
    """
    # ...

def insert_br_at_third_width(text, char_widths_dict: Dict[str, int]) -> str:
    """
    स्ट्रिंग की चौड़ाई को तीन बराबर भागों में विभाजित करता है, और 1/3 और 2/3 स्थानों पर <br> डालता है
    """
    # ...
```

इन फंक्शंस का उपयोग रूबी एनोटेशन के लिए पाठ के लेआउट को नियंत्रित करने के लिए किया जाता है, खासकर जब मूल पाठ और रूबी के अनुपात असमान होते हैं:

- **measure_text_width_Arial16**: टेक्स्ट की चौड़ाई मापने के लिए प्रत्येक अक्षर की पिक्सेल चौड़ाई जोड़ता है
- **insert_br_at_half_width**: बहुत लंबे रूबी टेक्स्ट को आधे में विभाजित करने के लिए उपयोगी
- **insert_br_at_third_width**: अत्यधिक लंबे रूबी टेक्स्ट को तीन भागों में विभाजित करता है

### 4.3 आउटपुट प्रारूप

```python
def output_format(main_text, ruby_content, format_type, char_widths_dict):
    """
    एस्पेरांतो शब्दमूल (main_text) और उसके अनुवाद/काञ्जी (ruby_content) को
    निर्दिष्ट format_type में जोड़ता है
    """
    if format_type == 'HTML格式_Ruby文字_大小调整':
        width_ruby = measure_text_width_Arial16(ruby_content, char_widths_dict)
        width_main = measure_text_width_Arial16(main_text, char_widths_dict)
        ratio_1 = width_ruby / width_main
        if ratio_1 > 6:
            return f'<ruby>{main_text}<rt class="XXXS_S">{insert_br_at_third_width(ruby_content, char_widths_dict)}</rt></ruby>'
        elif ratio_1 > (9/3):
            return f'<ruby>{main_text}<rt class="XXS_S">{insert_br_at_half_width(ruby_content, char_widths_dict)}</rt></ruby>'
        # और कई अन्य अनुपात केस...

    elif format_type == 'HTML格式_Ruby文字_大小调整_汉字替换':
        # मुख्य और रूबी की भूमिकाओं को उल्टा करने जैसा प्रारूप
        # ...

    elif format_type == 'HTML格式':
        return f'<ruby>{main_text}<rt>{ruby_content}</rt></ruby>'

    elif format_type == 'HTML格式_汉字替换':
        return f'<ruby>{ruby_content}<rt>{main_text}</rt></ruby>'

    elif format_type == '括弧(号)格式':
        return f'{main_text}({ruby_content})'

    elif format_type == '括弧(号)格式_汉字替换':
        return f'{ruby_content}({main_text})'

    elif format_type == '替换后文字列のみ(仅)保留(简单替换)':
        return f'{ruby_content}'
```

यह फंक्शन विभिन्न आउटपुट प्रारूपों के लिए प्रतिस्थापन फॉर्मेटिंग लॉजिक का मुख्य केंद्र है:

1. **HTML रूबी (आकार समायोजन के साथ)**: मुख्य पाठ के ऊपर अनुवाद दिखाता है, चौड़ाई के अनुपात के आधार पर फॉन्ट साइज और लाइन ब्रेक्स समायोजित करता है
2. **HTML रूबी (आकार समायोजन और काञ्जी के साथ)**: उपरोक्त जैसा, लेकिन मुख्य पाठ और अनुवाद की भूमिकाएं उलट जाती हैं
3. **बेसिक HTML**: बेसिक HTML रूबी मार्कअप
4. **कोष्ठक प्रारूप**: अनुवाद को कोष्ठकों में दिखाता है
5. **केवल प्रतिस्थापित पाठ**: केवल अनुवादित अंश दिखाता है

### 4.4 मल्टीप्रोसेसिंग फंक्शन्स

```python
def process_chunk_for_pre_replacements(
    chunk: List[List[str]],
    replacements: List[Tuple[str, str, str]]
) -> Dict[str, List[str]]:
    """
    chunk: [[E_root, pos], ...] पार्शियल लिस्ट
    safe_replace द्वारा प्रतिस्थापन परिणाम { E_root: [replaced_stem, pos], ... } के रूप में रिटर्न करता है
    """
    # ...

def parallel_build_pre_replacements_dict(
    E_stem_with_Part_Of_Speech_list: List[List[str]],
    replacements: List[Tuple[str, str, str]],
    num_processes: int = 4
) -> Dict[str, List[str]]:
    """
    डेटा को num_processes हिस्सों में विभाजित करता है, process_chunk_for_pre_replacements को पैरेलल चलाता है
    और अंत में शब्दकोशों को मर्ज करके रिटर्न करता है।
    """
    # ...
```

ये फंक्शन्स प्रतिस्थापन शब्दकोश बनाने की प्रक्रिया को पैरेलल में संचालित करते हैं, विशेष रूप से बड़े डेटासेट के लिए प्रदर्शन में सुधार करते हैं।

### 4.5 समान रूबी निकालना

```python
IDENTICAL_RUBY_PATTERN = re.compile(r'<ruby>([^<]+)<rt class="XXL_L">([^<]+)</rt></ruby>')

def remove_redundant_ruby_if_identical(text: str) -> str:
    """
    <ruby>xxx<rt class="XXL_L">xxx</rt></ruby> के जैसे,
    अगर पैरेंट स्ट्रिंग और रूबी स्ट्रिंग पूरी तरह से समान हैं तो <ruby> को हटा देता है
    """
    # ...
```

यह फंक्शन अनावश्यक रूबी एनोटेशन को हटाकर आउटपुट को अनुकूलित करता है, जहां र
