# Conceptual-Metaphor1
from google.colab import drive
drive.mount('/content/drive')

!pip install openai==0.28

import openai
import pandas as pd
import os


# API 키 설정
openai.api_key = '###'

# 파일 경로 설정
input_path = "/content/drive/MyDrive/기타/전산언어학1/transcript.csv"
output_path = os.path.join("/content/drive/MyDrive/기타/전산언어학1", "transcript_CC결과_2차.csv")

# CSV 파일 불러오기
data = pd.read_csv(input_path)

# 특정 발화자 행 필터링 ('VICE PRESIDENT KAMALA HARRIS:' 또는 'FORMER PRESIDENT DONALD TRUMP:'로 시작)
filtered_data = data[(data['transcript'].str.startswith('VICE PRESIDENT KAMALA HARRIS:')) | (data['transcript'].str.startswith('FORMER PRESIDENT DONALD TRUMP:'))]

# 결과 저장을 위한 칼럼 생성
filtered_data['Processed Text'] = ''
filtered_data['Conceptual Metaphor'] = ''
filtered_data['TD'] = ''
filtered_data['SD'] = ''


# ChatGPT API 요청 함수
def get_chatgpt_response(prompt):
    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=[{"role": "user", "content": prompt}]
    )
    return response['choices'][0]['message']['content']

# 단계별 데이터 처리
for index, row in filtered_data.iterrows():
    text = row['transcript']

    # 1. 개념적 은유가 사용된 문장 찾기
    prompt_1 = f"다음 문장에서 개념적 은유가 사용된 부분을 찾아주세요:\n{text}"
    conceptual_metaphor = get_chatgpt_response(prompt_1)

    # 2. 'Processed Text' 컬럼에 개념적 은유가 있는 문장만 굵게 표시하여 추가
    if conceptual_metaphor:
        processed_text = text.replace(conceptual_metaphor, f"**{conceptual_metaphor}**")
        filtered_data.at[index, 'Processed Text'] = processed_text

        # 3. 'Conceptual Metaphor' 컬럼에 대문자 및 단수 형태로 저장
        filtered_data.at[index, 'Conceptual Metaphor'] = conceptual_metaphor.upper()

        # 4. TD 및 SD로 개념적 은유 나누기
        prompt_2 = f"다음 개념적 은유에서 대상(TD)과 출처(SD)를 나누어주세요:\n{conceptual_metaphor}"
        td_sd_response = get_chatgpt_response(prompt_2)

        # TD와 SD가 올바르게 구분되었는지 확인
        if ',' in td_sd_response:
            td, sd = td_sd_response.split(',', 1)  # 첫 번째 쉼표로 나눔
            filtered_data.at[index, 'TD'] = td.strip()
            filtered_data.at[index, 'SD'] = sd.strip()
        else:
            # 에러 처리: TD와 SD로 분리하지 못한 경우
            filtered_data.at[index, 'TD'] = "분리 실패"
            filtered_data.at[index, 'SD'] = "분리 실패"
            filtered_data.at[index, '기타'] = f"TD/SD 구분 오류: {td_sd_response}"
            continue  # 다음 행으로 넘어가기



# 결과 CSV 파일로 저장
filtered_data.to_csv(output_path, index=False)




# Conceptual-Metaphor2


from google.colab import drive
drive.mount('/content/drive')

!pip install openai==0.28

#6차_챗GPT5=> 최종 > 논문 정리

import openai
import pandas as pd
import os
import inflect  # 단수/복수 형태를 변환하는 데 사용하는 라이브러리

# inflect 엔진 초기화
p = inflect.engine()


# API 키 설정
openai.api_key = '###'

# 파일 경로 설정
input_path = "/content/drive/MyDrive/기타/전산언어학1/transcript_CC결과_2차_분석.csv"
output_path = os.path.join("/content/drive/MyDrive/기타/전산언어학1", "transcript_개념화표현_완료.csv")

# CSV 파일 불러오기
data = pd.read_csv(input_path, encoding='EUC-KR')

# 한국어를 영어로 번역하는 함수
def translate_to_english(text):
    try:
        response = openai.ChatCompletion.create(
            model="gpt-4",
            messages=[{"role": "system", "content": "Please translate the following Korean text into English. Use singular, uppercase words."},
                      {"role": "user", "content": text}]
        )
        translation = response['choices'][0]['message']['content'].strip().upper()
        return p.singular_noun(translation) or translation  # Ensure the translation is singular
    except Exception as e:
        print(f"Translation error: {e}")
        return None

# 개념화 문장 생성 함수
def generate_metaphor(td, sd):
    try:
        td_english = translate_to_english(td)
        sd_english = translate_to_english(sd)
        if not td_english or not sd_english:
            print(f"Missing translation for TD: {td} or SD: {sd}")
            return None
        # Generate the metaphor using strict template to avoid hallucination
        metaphor = f"{td_english} IS {sd_english}"
        return metaphor
    except Exception as e:
        print(f"Error generating metaphor: {e}")
        return None

# 개념화 문장을 데이터프레임에 추가
data['conceptual_metaphor'] = data.apply(lambda row: generate_metaphor(row['TD'], row['SD']), axis=1)

# 결과 파일로 저장
data.to_csv(output_path, index=False, encoding='utf-8-sig')



