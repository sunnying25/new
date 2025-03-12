import sys
import time
import pickle
from PyQt5.QtWidgets import (QApplication, QWidget, QVBoxLayout, QLabel, QCalendarWidget,
                             QPushButton, QTextEdit, QGroupBox, QCheckBox, QHBoxLayout)
from PyQt5.QtGui import QIcon
from PyQt5.QtCore import QDateTime, QTimer, QThread, pyqtSignal, QEventLoop
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from webdriver_manager.chrome import ChromeDriverManager
from selenium.webdriver.common.action_chains import ActionChains
from PyQt5.QtCore import QTimer, QTime

class ReservationThread(QThread):
    log_signal = pyqtSignal(str)

    def __init__(self, selected_dates, selected_courts, selected_times):
        super().__init__()
        self.selected_dates = selected_dates
        self.selected_courts = selected_courts
        self.selected_times = selected_times

    def run(self):
        options = webdriver.ChromeOptions()
        # options.add_argument("--headless")
        options.add_argument("--disable-gpu")
        options.add_argument("--no-sandbox")
        options.add_argument("--disable-dev-shm-usage")
        # driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=options)
        # self.log_signal.emit("Selenium 드라이버 실행 완료.")

        for court in self.selected_courts:
            for date in self.selected_dates:
                for time_slot in self.selected_times:
                    success = self.make_reservation(date, time_slot, court)
                    if success:
                        self.log_signal.emit(f"✅ {date} {time_slot} {court} 예약 성공!")
                    else:
                        self.log_signal.emit(f"❌ {date} {time_slot} {court} 예약 실패.")
        # driver.quit()
        

    def make_reservation(self, selected_date, time_slot, court):
        """예약 매크로 실행"""
        month = int(selected_date.split('-')[1])
        date = int(selected_date.split('-')[2])       

        day_night = time_slot[:2]
        start_time = time_slot[2:]
        
        self.log_signal.emit(f"예약 매크로를 실행합니다.")

        options = webdriver.ChromeOptions()
        # options.add_argument("--headless")  # headless 모드 활성화
        driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=options)
        try:
            
            # 5. 저장된 쿠키 불러오기
            # 먼저 네이버 메인 페이지로 이동 (쿠키 도메인과 일치하도록)
            driver.get("https://www.naver.com")
            
            # 쿠키 추가
            with open("naver_cookies.pkl", "rb") as file:
                cookies = pickle.load(file)
            for cookie in cookies:
                driver.add_cookie(cookie)
            driver.refresh()
            
            
            # 8. 네이버 지도 페이지로 이동
            target_url = "https://map.naver.com/p/search/%EB%82%B4%EA%B3%A1%20%ED%85%8C%EB%8B%88%EC%8A%A4%EC%9E%A5/place/1257947832?placePath=?entry=pll&from=nx&fromNxList=true&searchType=place&c=15.00,0,0,0,dh"
            driver.get(target_url)
            
            
            # iframe 으로 포커스 이동
            time.sleep(1.5)
            driver.switch_to.frame(driver.find_element(By.ID,'entryIframe'))
            
            
            reserve_tab = WebDriverWait(driver, 5).until(
                EC.element_to_be_clickable((By.XPATH, "//a[span[text()='예약']]"))
            )
            reserve_tab.click()
            
            button = WebDriverWait(driver, 5).until(
                EC.element_to_be_clickable((By.XPATH, "//a[span[text()='더보기']]"))
            )
            button.click()
            
            # 코트 설정정
            xpath =  f"//*[contains(@class, 'COOCz') and starts-with(., '{month}') and contains(., '{court}')]"
            element = WebDriverWait(driver, 5).until(
                EC.element_to_be_clickable((By.XPATH, xpath)))

            try:
                time.sleep(1.5)
                driver.execute_script("arguments[0].scrollIntoView({block: 'center'});", element)
                element.click()
            except:
                time.sleep(1.5)
                driver.execute_script("arguments[0].scrollIntoView({block: 'center'});", element)
                element.click()
            
                
            # 캘린더에서 날짜 설정
            xpath =  f"//button[@class='calendar_date']//span[text()='{date}']"
            element = WebDriverWait(driver, 5).until(
                EC.element_to_be_clickable((By.XPATH, xpath))
            )
            driver.execute_script("arguments[0].scrollIntoView({block: 'center'});", element)
            element.click()

            
            if '오후' == day_night:
                start_time += 12
            
            # 오전 6시부터 시작이므로 버튼 인덱스 계산
            button_index = start_time - 6
            buttons = driver.find_elements(By.CLASS_NAME, 'btn_time')
            
            # 해당 버튼 클릭
            if 0 <= button_index < len(buttons):
                button_to_click = buttons[button_index]
                ActionChains(driver).move_to_element(button_to_click).click().perform()
            
            # For the first button (Next button)
            next_button = WebDriverWait(driver, 5).until(
                EC.element_to_be_clickable((By.XPATH, '//button[@class="footer_btn" and @data-click-code="nextbuttonview.request"]'))
            )
            driver.execute_script("arguments[0].scrollIntoView({block: 'center'});", next_button)
            driver.execute_script("arguments[0].scrollIntoView({block: 'center'});", next_button)
            next_button.click()
            
            # For the second button (Submit and agree button)
            submit_button = WebDriverWait(driver, 5).until(
                EC.element_to_be_clickable((By.XPATH, '//button[@class="btn_request" and @data-click-code="submitbutton.submit"]'))
            )
            driver.execute_script("arguments[0].scrollIntoView({block: 'center'});", submit_button)
            driver.execute_script("arguments[0].scrollIntoView({block: 'center'});", next_button)
            submit_button.click()
            
            ## 창 전환
            current_window = driver.current_window_handle
            WebDriverWait(driver, 5).until(EC.number_of_windows_to_be(2)) # 대기
            for handle in driver.window_handles:
                if handle != current_window:
                    driver.switch_to.window(handle)
                    print('전환완료')
                    break
            
            # 요소 찾기
            element = WebDriverWait(driver, 5).until(
                EC.presence_of_element_located((By.XPATH, "//*[@id='PAYMENT_WRAP']/div[1]/button"))
            )
            
            # 스크롤을 해당 요소로 완전히 내리기
            driver.execute_script("arguments[0].scrollIntoView({block: 'center'});", element)
            driver.execute_script("window.scrollBy(0, 400);")
            
            # 클릭 실행
            element.click()
            element.click()
            
            # 일반 결제 
            driver.execute_script("window.scrollBy(0, 600);")
            element = WebDriverWait(driver, 5).until(
                EC.element_to_be_clickable((By.XPATH, "//*[@id='PAYMENT_WRAP']/div[2]/div[4]/div/div/label/span"))
            )
            element.click()
            
            # 무통장결제
            element = WebDriverWait(driver, 5).until(
                EC.element_to_be_clickable((By.CLASS_NAME, "SolidTabFlexible_button-tab__FGd0S"))
            )
            driver.execute_script("arguments[0].scrollIntoView({block: 'center'});", element)
            element.click()
            
            
            # 무통장결제
            element = WebDriverWait(driver, 5).until(
                EC.element_to_be_clickable((By.XPATH, "//button[contains(text(), '무통장입금')]"))
            )
            driver.execute_script("arguments[0].scrollIntoView({ block: 'center'});", element)
            element.click()
        
            
            #은행선택위한 클릭
            element = WebDriverWait(driver, 5).until(
                EC.element_to_be_clickable((By.XPATH, '//*[@id="PAYMENT_WRAP"]/div[2]/div[4]/div[3]/div[1]/div/button'))
            )
            driver.execute_script("arguments[0].scrollIntoView({ block: 'center'});", element)
            element.click()
            
            # 우리 은행
            element = WebDriverWait(driver, 5).until(
                EC.element_to_be_clickable((By.XPATH, "//span[text()='우리은행']"))
            )
            driver.execute_script("arguments[0].scrollIntoView({block: 'center'});", element)
            element.click()


            # 환급 방법 설정
            element = WebDriverWait(driver, 5).until(
                EC.element_to_be_clickable((By.XPATH,'//*[@id="REFUND_ACCOUNT_SECTION"]/ul/li[2]/div/span/label'))
            )
            driver.execute_script("arguments[0].scrollIntoView({block: 'center'});", element)
            element.click()

            # 결제
            element = WebDriverWait(driver, 5).until(
                EC.element_to_be_clickable((By.XPATH, '//*[@id="root"]/div/div[2]/div[5]/div/div/div[2]/button'))
            )
            element.click()
            
            driver.quit()
    
            self.log_signal.emit(f"{_dates, day_night, start_time, court} 예약 완료, 무통장 입금 계좌를 네이버 계정으로 확인하세요.")
            return True
        except Exception as e:
            driver.quit()
            return False
            

class MacroGUI(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("내곡 테니스장 예약 매크로")
        self.setWindowIcon(QIcon('icon.png'))
        self.setGeometry(100, 100, 800, 500)
        layout = QVBoxLayout()

        # 날짜 선택
        self.calendar = QCalendarWidget(self)
        self.calendar.clicked.connect(self.on_date_click)
        layout.addWidget(self.calendar)
        self.selected_dates_text = QTextEdit(self)
        self.selected_dates_text.setReadOnly(True)
        layout.addWidget(self.selected_dates_text)
        self.selected_dates = []

        # 코트 선택
        self.court_group = self.create_court_selection_group()
        layout.addWidget(self.court_group)

        # 시간 선택
        self.time_group = self.create_time_selection_group()
        layout.addWidget(self.time_group)

        # 실행 버튼
        self.run_button = QPushButton("예약 실행", self)
        self.run_button.clicked.connect(self.check_time)
        layout.addWidget(self.run_button)

        # 로그인 완료
        self.login_button = QPushButton("로그인 완료", self)
        self.login_button.clicked.connect(self.on_login_complete)
        layout.addWidget(self.login_button)
        self.login_event_loop = QEventLoop()

        # 로그 출력
        self.log_text = QTextEdit(self)
        self.log_text.setReadOnly(True)
        self.log_text.setStyleSheet("background-color: black; color: white;")
        layout.addWidget(QLabel("로그 출력:"))
        layout.addWidget(self.log_text)

        # 예약 실행할 시간 설정 (예: 08:00)
        self.target_time = "23:50"
        
        self.setLayout(layout)

    def create_court_selection_group(self):
        group = QGroupBox("코트 선택")
        layout = QHBoxLayout()
        self.court_checkboxes = {}
        for court in ["1번코트", "2번코트", "3번코트"]:
            checkbox = QCheckBox(court)
            layout.addWidget(checkbox)
            self.court_checkboxes[court] = checkbox
        group.setLayout(layout)
        return group

    def create_time_selection_group(self):
        group = QGroupBox("시간 선택")
        layout = QHBoxLayout()
        self.time_checkboxes = {}
        for time in ['오전6시', '오전7시', '오전8시', '오전9시', '오전10시', '오전11시', '오후12시', '오후1시', '오후2시', '오후3시', '오후4시', '오후5시', '오후6시', 
                     '오후7시','오후8시', '오후9시']:
            checkbox = QCheckBox(f"{time}")
            layout.addWidget(checkbox)
            self.time_checkboxes[f"{time}"] = checkbox
        group.setLayout(layout)
        return group

    def on_login_complete(self):
        """로그인 완료 버튼 클릭 시 호출"""
        self.append_log("로그인 완료 버튼 클릭됨")
        self.login_event_loop.quit()  # 이벤트 루프 종료

    def on_date_click(self, date):
        date_str = date.toString("yyyy-MM-dd")
        if date_str in self.selected_dates:
            self.selected_dates.remove(date_str)
        else:
            self.selected_dates.append(date_str)
        self.selected_dates_text.setPlainText("\n".join(self.selected_dates))

    def append_log(self, message):
        self.log_text.append(f"[{time.strftime('%H:%M:%S')}] {message}")

    def save_login_info(self):
        ''' 로그인 계정 생성'''
        options = webdriver.ChromeOptions()
        driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=options)

        # 네이버 로그인 페이지로 이동
        driver.get("https://nid.naver.com/nidlogin.login?mode=qrcode&url=https%3A%2F%2Fwww.naver.com%2F&locale=ko_KR&svctype=1&loginitp=null")
        
        # 로그인 완료 버튼을 누를 때까지 대기
        self.append_log("로그인 후 '로그인 완료' 버튼을 눌러주세요.")
        self.login_event_loop.exec_()  # 이벤트 루프 실행 (로그인 완료 버튼 클릭까지 대기)

        # 2. 쿠키 저장
        cookies = driver.get_cookies()
        with open("naver_cookies.pkl", "wb") as file:
            pickle.dump(cookies, file)
        self.append_log("쿠키 저장 완료") 
        driver.quit()


    def run_macro(self):
        self.save_login_info() #로그인정보
        
        selected_courts = [court for court, checkbox in self.court_checkboxes.items() if checkbox.isChecked()]
        selected_times = [time for time, checkbox in self.time_checkboxes.items() if checkbox.isChecked()]
        if not self.selected_dates or not selected_courts or not selected_times:
            self.append_log("❌ 날짜, 코트, 시간을 모두 선택해주세요.")
            return

        
        self.append_log("⏳ 예약 실행 중...")
        self.thread = ReservationThread(self.selected_dates, selected_courts, selected_times)
        self.thread.log_signal.connect(self.append_log)
        self.thread.start()


if __name__ == '__main__':
    app = QApplication(sys.argv)
    gui = MacroGUI()
    gui.show()
    sys.exit(app.exec_())
특정 시간이 되면 thread가 run하도록
