import requests
import json 
import time 
import pandas as pd
     
Документация по Api HH: https://github.com/hhru/api

Для использования методов, требующих авторизацию пользователя или приложения, вам необходимо зарегистрировать приложение по адресу https://dev.hh.ru и настроить процесс авторизации.

Зарегистрированное приложение может запрашивать у пользователей hh.ru разрешение доступа к их персональным данным, без получения и хранения их логина и пароля.

В нашем примере такие методы это поиск id резюме и рассылка резюме. Поиск вакансий не требует регистрации.

Поиск вакансии


# Функция вытаскивает вакансии по параметрам (постронично)
def get_vacancies_per_page(page: int, search_row: str):
    params = {
        'text':search_row, 
        'area':1, # регион поиска - Москва
        'page':page, 
        'per_page': 100 # количество вакансий на странице
        }

    request_url = 'https://api.hh.ru/vacancies'
    request = requests.get(request_url, params=params)
    request_content = request.content.decode()
    request.close()
    return request_content
     

# Нахождение вакансий по поисковой строке 
def get_all_vacancies(search_row):
    data = []
    for page in range(0, 20):
        page_data = json.loads(get_vacancies_per_page(page=page, search_row = search_row))
        data.extend(page_data['items'])

        if(page_data['pages'] - page) <=1:
            break

        time.sleep(0.25)
    df_vacancies = pd.DataFrame(data)    
    start = "{'id': '"
    end = "', 'name':"
    df_clean = df_vacancies
    df_clean['experience'] = df_clean['experience'].astype(str)
    df_clean['experience'] = df_clean['experience'].apply(lambda x: x[x.find(start)+len(start):x.rfind(end)])  
    return df_clean
     

# Определяем поисковую строку 
# Больше примеров можно посмотреть на https://hh.ru/article/1175
search_row = f'NAME:("data analyst" OR "аналитик данных" OR "data аналитик")'
     

# Делаем запрос
df_vacancies = get_all_vacancies(search_row)
     

# Мы можем работать с результатом поиска 
# Например, посмотреть сколько вакансий по требуемому опыту 
df_groupby_experience = df_vacancies.groupby('experience')['id'].count()
df_groupby_experience
     

# Или сфильтровать вакансии с опытом и без 
df_with_salary = df_vacancies[df_vacancies['salary'].notna()]
df_no_salary = df_vacancies[df_vacancies['salary'].isna()]
     
Рассылка резюме

Гайд по прегистрации от Учимся вместе: https://youtu.be/m1hzdcYxs4M?t=1183 Также в этом видео в общем показывается как работать с hh.


# Для методов требующих авторизацию
authorization_code = ""
Client_ID = ""
Client_Secret = ""
     

# Определяем параметры
params = {
    'grant_type':'authorization_code',
    'client_id':Client_ID,
    'client_secret':Client_Secret,
    'code':authorization_code}
     

# Находим access Token 
result = json.loads(requests.post(f'https://hh.ru/oauth/token', params=params)
                          .content.decode())
access_token = result['access_token']
     

# Определяем загаловок (не забудьте поставить ваш мейл, если хотите)
headers = {
        'Authorization': f'Bearer {access_token}',
        'HH-User-Agent': 'Send Resume (your_mail@mail.ru)'
}
     
Находим ID RESUME

Вы можете найти ID Вашего резюме в ручную - зайдите на сраницу резюме, id resume будет иди после resume. https://hh.ru/resume/ID_RESUME


# Находим ID resume через запрос
resume_list = json.loads(
    requests.get('https://api.hh.ru/resumes/mine', headers = headers).content.decode()
)['items']
df_resume_list = pd.DataFrame(resume_list) 
df_resume_ids = df_resume_list[['title', 'id']]
df_resume_ids
     

# Выбираете id resume из списка либо вручную 
resume_id = ""
     

# Определяем vacancies id 
vacancies_id = df_vacancies['id'].to_list()
     

# Сопроводительное письмо 
message = ""
     

# Отправка Одного Резюме 
def send_resume(vacancy_id, resume_id, message):
    params = {
            'vacancy_id':vacancy_id, 
            'resume_id':resume_id, 
            "message":message 
            }

    click_url = 'https://api.hh.ru/negotiations'
    requests.post(click_url, headers = headers, params=params)
     

# Простой пример функции для массовой отправки
def send_all_resume(vacancies_id, resume_id, message):
    for id in vacancies_id:
        send_resume(
        vacancy_id=id,
        resume_id=resume_id,
        message=message
    )
     
BONUS - открытие в chrome браузере вакансий для Windows. Желательно открывать частями. Открывается в последнем открытом браузере.


import os
current_file = os.path.abspath('')
import webbrowser 
# getting path 
chrome_path = "C://Program Files (x86)//Google//Chrome//Application//Chrome.exe %s"
  
# First registers the new browser 
webbrowser.register('chrome', None,  
                    webbrowser.BackgroundBrowser(chrome_path)) 
def open_vacancies(list):
    for id in list:
        url = "https://hh.ru/vacancy/"
        webbrowser.get(chrome_path).open_new(url+id) 
     
