[app.py](https://github.com/user-attachments/files/31451172/app.py)

[config]
SCM_DO_BUILD_DURING_DEPLOYMENT=true 
.git
.gitignore
.vscode
__pycache__
*.pyc
.env
*.env
.DS_Store[Uploading bot.py…]()

README.md
 
# Secrets
*.env
.env
.env.*

# Python
__pycache__/
*.py[cod]
.venv/
venv/

# OS / editor
.DS_Store
.vscode/
 
import os
import sys
import traceback
import base64

try:
    import pyfiglet
except ImportError:
    pyfiglet = None

from aiohttp import web
from botbuilder.core import BotFrameworkAdapter, BotFrameworkAdapterSettings
from botbuilder.core.integration import aiohttp_error_middleware
from botbuilder.schema import Activity
from bot import MyTeamsBot
 
# ------------------------------------------------------------
print("\n" + "="*70, flush=True)
if pyfiglet:
    try:
        _sig = base64.b64decode("RG9taW5pYzkx").decode('utf-8')
        ascii_banner = pyfiglet.figlet_format(_sig, font="slant")
        print(ascii_banner, flush=True)
        print(f"🚀 System Kernel Loaded. Architecture by {_sig}.", flush=True)
    except:
        print("System Loaded.", flush=True)
else:
    print("⚠️ 'pyfiglet' library not found.", flush=True)
print("="*70 + "\n", flush=True)
# ------------------------------------------------------------

# [1] 환경 변수
APP_ID = os.environ.get("MicrosoftAppId", "")
APP_PASSWORD = os.environ.get("MicrosoftAppPassword", "")
TENANT_ID = os.environ.get("MicrosoftAppTenantId")

# [2] 어댑터 설정
if TENANT_ID:
    print(f"🔒 Single Tenant Mode (Tenant: {TENANT_ID})", flush=True)
    settings = BotFrameworkAdapterSettings(app_id=APP_ID, app_password=APP_PASSWORD, channel_auth_tenant=TENANT_ID)
else:
    print("🌐 Multi Tenant Mode", flush=True)
    settings = BotFrameworkAdapterSettings(APP_ID, APP_PASSWORD)

adapter = BotFrameworkAdapter(settings)

# 3. 봇 생성
bot = MyTeamsBot()
bot.adapter = adapter 

# [NEW] 앱 시작 시 스케줄러 실행 (이벤트 루프 동기화)
async def on_startup(app):
    print("🚀 App Service Started. Initializing Scheduler...", flush=True)
    bot.start_scheduler()

# 4. 핸들러
async def messages(req: web.Request) -> web.Response:
    if "application/json" in req.headers.get("Content-Type", ""):
        body = await req.json()
    else:
        return web.Response(status=415)

    activity = Activity().deserialize(body)
    auth_header = req.headers.get("Authorization", "")

    try:
        response = await adapter.process_activity(activity, auth_header, bot.on_message_activity)
        if response:
            return web.json_response(data=response.body, status=response.status)
        return web.Response(status=201)
    except Exception as e:
        print(f"❌ Error: {e}", flush=True)
        traceback.print_exc()
        return web.Response(status=500, text=str(e))

app = web.Application(middlewares=[aiohttp_error_middleware])
app.router.add_post("/api/messages", messages)

# [NEW] 시작 이벤트 등록
app.on_startup.append(on_startup)

if __name__ == "__main__":
    try:
        port = int(os.environ.get("PORT", 8000))
        print(f"Listening on port {port}...", flush=True)
        web.run_app(app, host="0.0.0.0", port=port)
    except Exception as e:
        print(f"Startup Failed: {e}", flush=True)
        raise e
import traceback
import json
from botbuilder.core import ActivityHandler, TurnContext, CardFactory
from botbuilder.schema import Activity, Attachment, ActionTypes, CardAction, HeroCard, ActivityTypes
from datetime import datetime, timedelta
from apscheduler.schedulers.asyncio import AsyncIOScheduler
import base64
import os
import io
import pandas as pd
import requests
import openai_helper
import graph_helper
import report_helper
 
# 스토리지 관련 라이브러리
try:
    from azure.storage.blob import BlobServiceClient, generate_blob_sas, BlobSasPermissions
except ImportError:
    BlobServiceClient = None

# 환경 변수
ADMIN_REPORT_EMAIL = os.environ.get("REPORT_ADMIN_EMAIL", "admin@yourdomain.com")
STORAGE_CONN_STR = os.environ.get("STORAGE_CONNECTION_STRING", "") 

class MyTeamsBot(ActivityHandler):
    def __init__(self):
        super().__init__()
        self._sig = base64.b64decode("ZG9taW5pYzkx").decode('utf-8')
        
        # [상태 관리]
        self.user_state = {}
        self.user_pending_work = {}
        self.audit_logs = []
        
        # [스케줄러 설정] 매주 금요일 오후 5시 (17:00)
        self.scheduler = AsyncIOScheduler()
        self.scheduler.add_job(self.scheduled_cost_report, 'cron', day_of_week='fri', hour=17, minute=0)
        
        self.last_ref = None
        self.last_user_id = None 

    def start_scheduler(self):
        if not self.scheduler.running:
            self.scheduler.start()
            print("⏰ Weekly Scheduler started (Friday 17:00)", flush=True)

    def _df_to_markdown(self, df):
        try:
            if df.empty: return "(데이터 없음)"
            headers = df.columns.tolist()
            md = "| " + " | ".join(str(h) for h in headers) + " |\n"
            md += "| " + " | ".join(["---"] * len(headers)) + " |\n"
            for _, row in df.iterrows():
                row_str = "| " + " | ".join(str(v).replace('\n', ' ') for v in row.values) + " |\n"
                md += row_str
            return md
        except:
            return df.to_string()

    # 실시간 환율 조회
    def get_realtime_exchange_rate(self):
        try:
            url = "https://api.exchangerate-api.com/v4/latest/USD"
            resp = requests.get(url, timeout=5)
            if resp.status_code == 200:
                data = resp.json()
                rate = float(data['rates']['KRW'])
                return rate
        except Exception as e:
            print(f"⚠️ Exchange Rate API Error: {e}")
        return 1450.0 

    def _upload_and_get_link(self, file_bytes, file_name):
        if not STORAGE_CONN_STR or not BlobServiceClient:
            print("❌ 스토리지 설정이 없습니다. (STORAGE_CONNECTION_STRING 확인 필요)")
            return None
        
        try:
            container_name = "bot-reports"
            blob_service_client = BlobServiceClient.from_connection_string(STORAGE_CONN_STR)
            
            container_client = blob_service_client.get_container_client(container_name)
            if not container_client.exists():
                container_client.create_container()

            blob_client = container_client.get_blob_client(file_name)
            blob_client.upload_blob(file_bytes, overwrite=True)
            
            sas_token = generate_blob_sas(
                account_name=blob_client.account_name,
                container_name=container_name,
                blob_name=file_name,
                account_key=blob_service_client.credential.account_key,
                permission=BlobSasPermissions(read=True),
                expiry=datetime.utcnow() + timedelta(hours=1)
            )
            
            return f"{blob_client.url}?{sas_token}"
        except Exception as e:
            print(f"❌ Upload Error: {e}")
            return None

    async def on_message_activity(self, turn_context: TurnContext):
        self.last_ref = TurnContext.get_conversation_reference(turn_context.activity)
        raw_text = turn_context.activity.text or ""
        sender_id = turn_context.activity.from_property.aad_object_id
        sender_name = turn_context.activity.from_property.name
        self.last_user_id = sender_id 
        
        print(f"\n📩 [Msg] {sender_name}: {raw_text}", flush=True)

        if turn_context.activity.attachments:
            for att in turn_context.activity.attachments:
                if "text/html" not in (att.content_type or ""):
                    await self._handle_file_upload(turn_context, sender_id, att)
                    return

        if turn_context.activity.value:
            await self._handle_card_action(turn_context, turn_context.activity.value, sender_id, sender_name)
            return
            
        if sender_id not in self.user_state: 
            self.user_state[sender_id] = {'unauth_count': 0, 'security_count': 0}
            
        roles = await graph_helper.get_user_admin_roles(sender_id)
        res = await openai_helper.get_chat_decision(raw_text, roles)
        intent = res.get("type")

        if intent == "SECURITY_ALERT":
            self.user_state[sender_id]['security_count'] += 1
            cnt = self.user_state[sender_id]['security_count']
            await turn_context.send_activity(f"⚠️ [보안 경고] 허용되지 않은 질문입니다. (누적: {cnt}회)")
            return
        
        if intent == "AUDIT_QUERY":
            recent = self.audit_logs[-10:] if self.audit_logs else []
            if recent:
                df_log = pd.DataFrame(recent)
                table_md = self._df_to_markdown(df_log)
                await turn_context.send_activity(f"📋 **최근 작업 로그**\n\n{table_md}")
            else:
                await turn_context.send_activity("📭 로그가 없습니다.")
            return

        if intent == "WORK":
            target = res.get("target_email")
            detail = res.get("detail")
            act = res.get("action")
            cat = res.get("intent")
            
            await turn_context.send_activity(f"⚙️ 작업 수행 중... ({act} {detail})")
            msg = ""
            if act == "REMOVE":
                if cat == "ADMIN_ROLE": msg = await graph_helper.remove_directory_role_pim(target, detail)
                else: msg = await graph_helper.remove_license(target, detail)
            else:
                if cat == "ADMIN_ROLE": msg = await graph_helper.assign_directory_role_pim(target, detail)
                else: msg = await graph_helper.assign_license(target, detail)
            self.audit_logs.append({"Date": datetime.now().strftime("%Y-%m-%d %H:%M"), "User": sender_name, "Action": f"{act} {cat}", "Target": target})
            await turn_context.send_activity(msg)
            return

        await turn_context.send_activity(res.get("content"))

    async def _handle_file_upload(self, turn_context, sender_id, attachment):
        try:
            file_url = attachment.content.get("downloadUrl") if attachment.content else attachment.content_url
            token = graph_helper.get_access_token()
            r = requests.get(file_url, headers={"Authorization": f"Bearer {token}"})
            if r.status_code != 200:
                await turn_context.send_activity("❌ 파일 다운로드 실패. 권한을 확인하세요.")
                return

            df_raw = pd.read_excel(io.BytesIO(r.content), header=None, nrows=20)
            header_idx = 0
            found = False
            for idx, row in df_raw.iterrows():
                row_str = " ".join([str(v) for v in row.values])
                if ("계정" in row_str and "작업" in row_str) or ("Choice1" in row_str) or ("License" in row_str and ("User" in row_str or "Email" in row_str)):
                    header_idx = idx
                    found = True
                    break
            
            df = pd.read_excel(io.BytesIO(r.content), header=header_idx if found else 0)
            new_cols = {}
            for c in df.columns:
                cs = str(c).strip()
                if "계정" in cs: new_cols[c] = "계정"
                elif "작업" in cs: new_cols[c] = "작업"
                elif "유형" in cs: new_cols[c] = "유형"
                elif "상세" in cs: new_cols[c] = "상세"
                else: new_cols[c] = cs
            df.rename(columns=new_cols, inplace=True)
            
            work_type = "general_work"
            if 'Choice1' in df.columns: work_type = 'create_user'
            elif 'License' in df.columns: work_type = 'assign_license'
            
            self.user_pending_work[sender_id] = {'df': df, 'type': work_type}
            preview_df = df.head(5).fillna("")
            table_md = self._df_to_markdown(preview_df)
            await turn_context.send_activity(f"📋 **Excel 내용 미리보기** ({work_type})\n\n{table_md}")
            
            card = HeroCard(
                title="작업 승인 요청",
                text="위 내용을 확인하셨다면 작업을 승인해주세요.",
                buttons=[
                    CardAction(type=ActionTypes.message_back, title="✅ 승인", text="APPROVE_BULK", value={"action": "approve_bulk"}),
                    CardAction(type=ActionTypes.message_back, title="❌ 거부", text="REJECT_BULK", value={"action": "reject_bulk"})
                ]
            )
            await turn_context.send_activity(Activity(type=ActivityTypes.message, attachments=[CardFactory.hero_card(card)]))

        except Exception as e:
            traceback.print_exc()
            await turn_context.send_activity(f"❌ 파일 처리 오류: {e}")

    async def _handle_card_action(self, turn_context, value, sender_id, sender_name):
        action = value.get("action")
        
        if action == "refresh_report":
            await turn_context.send_activity("🔄 리포트를 갱신합니다 (Power BI Style)...")
            await self.scheduled_cost_report() 
            return

        if action == "reject_bulk":
            self.user_pending_work.pop(sender_id, None)
            await turn_context.send_activity("❌ 작업이 취소되었습니다.")
            return

        if action == "approve_bulk":
            pending = self.user_pending_work.get(sender_id)
            if not pending:
                await turn_context.send_activity("⚠️ 요청이 만료되었습니다.")
                return
            
            roles = await graph_helper.get_user_admin_roles(sender_id)
            if not roles:
                await turn_context.send_activity("🚫 관리자 권한이 없어 작업을 수행할 수 없습니다.")
                return

            df = pending['df']
            work_type = pending['type']
            report_list = []

            await turn_context.send_activity(f"⚙️ 총 {len(df)}건의 작업을 시작합니다...")

            if work_type == 'create_user':
                for _, row in df.iterrows():
                    name = row.get('Name', 'Unknown')
                    dept = row.get('Team') or row.get('Department')
                    choices = [row.get('Choice1'), row.get('Choice2'), row.get('Choice3')]
                    final_upn = None
                    for c in choices:
                        if c and str(c).strip():
                            if not await graph_helper.check_user_exists(c):
                                final_upn = c
                                break
                    res_msg = ""
                    if not final_upn: res_msg = "❌ 중복 ID 불가"
                    else:
                        ok, msg = await graph_helper.create_user(final_upn, name, dept, "Staff")
                        res_msg = msg
                        if ok:
                            sku = "Office 365 E3" 
                            if dept and "Dev" in str(dept): sku = "Visual Studio"
                            l_res = await graph_helper.assign_license(final_upn, sku)
                            res_msg += f" / {l_res}"
                    report_list.append({"User": name, "ID": final_upn or "N/A", "Result": res_msg})

            elif work_type == 'assign_license':
                for _, row in df.iterrows():
                    email = row.get('User') or row.get('Email')
                    sku = row.get('License')
                    res_msg = "정보 부족"
                    if email and sku: res_msg = await graph_helper.assign_license(email, sku)
                    report_list.append({"User": email, "License": sku, "Result": res_msg})

            elif work_type == 'general_work':
                for _, row in df.iterrows():
                    email = row.get('계정')
                    act = row.get('작업')
                    cat = row.get('유형')
                    detail = row.get('상세')
                    if pd.isna(email) or pd.isna(act): continue
                    res_msg = "대기"
                    if act == '부여':
                        if cat == '라이선스': res_msg = await graph_helper.assign_license(email, detail)
                        elif cat == '관리자': res_msg = await graph_helper.assign_directory_role_pim(email, detail)
                    elif act == '회수':
                        if cat == '라이선스': res_msg = await graph_helper.remove_license(email, detail)
                        elif cat == '관리자': res_msg = await graph_helper.remove_directory_role_pim(email, detail)
                    else: res_msg = "Skip"
                    report_list.append({"User": email, "Action": f"{act} {cat}", "Detail": detail, "Result": res_msg})
            
            self.user_pending_work.pop(sender_id, None)
            if report_list:
                res_df = pd.DataFrame(report_list)
                table_md = self._df_to_markdown(res_df)
                await turn_context.send_activity(f"✅ **작업 완료 리포트**\n\n{table_md}")
            else:
                await turn_context.send_activity("✅ 완료되었으나 처리된 항목이 없습니다.")

    # 스케줄러: 비용 리포트 생성 (Power BI Style + Friday 17:00)
    async def scheduled_cost_report(self):
        if not self.last_ref: return
        adapter = getattr(self, 'adapter', None)
        if not adapter: return
        target_user_oid = self.last_user_id 
        if not target_user_oid: return
        app_id = adapter.settings.app_id

        async def callback(turn_context):
            print(f"\n⏰ [Weekly Report] Generating Power BI Style Report for {target_user_oid}...", flush=True)
            
            my_subs = await graph_helper.get_owner_subscriptions(target_user_oid)
            if not my_subs:
                await turn_context.send_activity("🔍 구독 정보를 찾을 수 없습니다.")
                return

            current_rate = self.get_realtime_exchange_rate()
            total_sum_krw = 0.0
            report_data = [] 
            
            # 1. 상세 데이터 수집 (리소스별, 지역별, RG별)
            for sub in my_subs:
                cost_info = await graph_helper.get_advanced_cost_data(sub['id'], timeframe="WeekToDate")
                if cost_info:
                    sub_total_krw = cost_info['total'] * current_rate
                    total_sum_krw += sub_total_krw
                    
                    # 환율 적용 헬퍼
                    def apply_rate(items):
                        return [{"name": i['name'], "cost": i['cost'] * current_rate, "type": i.get('type','')} for i in items]

                    report_data.append({
                        "sub_name": sub['name'],
                        "total": sub_total_krw,
                        "top_10_resources": apply_rate(cost_info['top_10_resources']),
                        "by_location": apply_rate(cost_info['by_location']),
                        "by_rg": apply_rate(cost_info['by_rg'])
                    })

            # 2. PDF/PPT 생성
            pdf_url, ppt_url = None, None
            if STORAGE_CONN_STR and report_data:
                pdf_bytes = report_helper.generate_cost_pdf(report_data)
                pdf_url = self._upload_and_get_link(pdf_bytes, f"WeeklyCost_{datetime.now().strftime('%Y%m%d')}.pdf")
                
                ppt_bytes = report_helper.generate_cost_ppt(report_data)
                ppt_url = self._upload_and_get_link(ppt_bytes, f"WeeklyCost_{datetime.now().strftime('%Y%m%d')}.pptx")

            # 3. Adaptive Card (Top 10 중심)
            card_body = [
                {"type": "TextBlock", "text": "📊 Weekly Azure Insight", "weight": "Bolder", "size": "Large", "color": "Accent"},
                {"type": "TextBlock", "text": f"1 USD = {current_rate:,.0f} KRW | {datetime.now().strftime('%Y-%m-%d %H:%M')}", "isSubtle": True}
            ]
            
            for sub in report_data:
                card_body.append({
                    "type": "Container", "separator": True, "spacing": "Medium",
                    "items": [
                        {"type": "TextBlock", "text": f"📂 {sub['sub_name']}", "weight": "Bolder"},
                        {"type": "TextBlock", "text": f"Total: ₩{sub['total']:,.0f}", "size": "Large", "color": "Attention"}
                    ]
                })

                # 차트 이미지 삽입 (Top 10 리소스)
                chart_bytes = report_helper.create_bar_chart(sub['top_10_resources'], title="Top 10 Resources")
                if chart_bytes:
                    b64_img = base64.b64encode(chart_bytes.getvalue()).decode("utf-8")
                    card_body.append({"type": "Image", "url": f"data:image/png;base64,{b64_img}", "size": "Stretch"})

                # Top 5 텍스트 나열
                fact_items = []
                for t in sub['top_10_resources'][:5]:
                    short_name = (t['name'][:25] + '..') if len(t['name']) > 25 else t['name']
                    fact_items.append({"title": short_name, "value": f"₩{t['cost']:,.0f}"})
                
                if fact_items:
                    card_body.append({"type": "FactSet", "facts": fact_items})

            card_body.append({
                "type": "Container", "separator": True,
                "items": [{"type": "TextBlock", "text": f"💎 Grand Total: ₩{total_sum_krw:,.0f}", "weight": "Bolder", "size": "ExtraLarge", "color": "Good"}]
            })

            actions = [{"type": "Action.Submit", "title": "🔄 Refresh Now", "data": {"action": "refresh_report"}}]
            if pdf_url: actions.append({"type": "Action.OpenUrl", "title": "📥 PDF Report", "url": pdf_url})
            if ppt_url: actions.append({"type": "Action.OpenUrl", "title": "📊 PPT Slides", "url": ppt_url})

            await turn_context.send_activity(Activity(
                type=ActivityTypes.message,
                attachments=[CardFactory.adaptive_card({"type": "AdaptiveCard", "version": "1.3", "body": card_body, "actions": actions})]
            ))

        try:
            await adapter.continue_conversation(self.last_ref, callback, bot_id=app_id)
        except Exception as e:
            print(f"❌ [Scheduler Error] {e}", flush=True)

    async def report_to_admin(self, message):
        print(f"[ADMIN] {message}", flush=True)
FROM python:3.11-slim-bookworm

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    g++ \
    pkg-config \
    libcairo2-dev \
    libfreetype6-dev \
    libpng-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PORT=8000
EXPOSE 8000

CMD ["python", "app.py"]
 
 
import os
import requests
import difflib
from azure.identity import ClientSecretCredential

# ---------------------------------------------------------
# [인증 및 토큰 발급]
# ---------------------------------------------------------
def get_access_token():
    try:
        cred = ClientSecretCredential(
            os.getenv("MicrosoftAppTenantId"), 
            os.getenv("MicrosoftAppId"), 
            os.getenv("MicrosoftAppPassword")
        )
        return cred.get_token("https://graph.microsoft.com/.default").token
    except Exception as e:
        print(f"❌ Token Error: {e}")
        return None
 
def get_arm_token():
    try:
        cred = ClientSecretCredential(
            os.getenv("MicrosoftAppTenantId"), 
            os.getenv("MicrosoftAppId"), 
            os.getenv("MicrosoftAppPassword")
        )
        return cred.get_token("https://management.azure.com/.default").token
    except:
        return None

# ---------------------------------------------------------
# [유틸리티: SKU 매칭, User ID 조회 등]
# ---------------------------------------------------------
def _find_best_sku_match(keyword, sku_list):
    clean_keyword = str(keyword).strip().replace(" ", "_").lower()
    clean_keyword = clean_keyword.replace("office", "microsoft") 
    sku_map = {s['skuPartNumber'].lower(): s for s in sku_list}
    all_names = list(sku_map.keys())
    for name in all_names:
        if clean_keyword in name: return sku_map[name]
    matches = difflib.get_close_matches(clean_keyword, all_names, n=1, cutoff=0.4)
    if matches: return sku_map[matches[0]]
    return None

async def _get_user_id(email, token):
    if not email: return None
    clean_id = str(email).strip().replace(" ", "")
    headers = {"Authorization": f"Bearer {token}"}
    resp = requests.get(f"https://graph.microsoft.com/v1.0/users?$filter=(mail eq '{clean_id}' or userPrincipalName eq '{clean_id}')&$select=id", headers=headers)
    if resp.status_code == 200 and resp.json().get('value'):
        return resp.json()['value'][0]['id']
    resp = requests.get(f"https://graph.microsoft.com/v1.0/users/{clean_id}?$select=id", headers=headers)
    if resp.status_code == 200: return resp.json().get('id')
    return None

async def _get_role_definition_id(role_name, token):
    headers = {"Authorization": f"Bearer {token}"}
    resp = requests.get(f"https://graph.microsoft.com/v1.0/roleManagement/directory/roleDefinitions?$filter=displayName eq '{role_name}'", headers=headers)
    if resp.status_code == 200 and resp.json().get('value'):
        return resp.json()['value'][0]['id']
    return None

# ---------------------------------------------------------
# [기능 1: PIM (관리자 권한) 관리]
# ---------------------------------------------------------
async def remove_directory_role_pim(email, role_name):
    try:
        token = get_access_token()
        uid = await _get_user_id(email, token)
        if not uid: return f"❌ 사용자 못 찾음: {email}"
        role_id = await _get_role_definition_id(role_name, token)
        if not role_id: return f"❌ 역할 이름 오류: '{role_name}'"
        headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
        payload = {"action": "adminRemove", "principalId": uid, "roleDefinitionId": role_id, "directoryScopeId": "/", "justification": "Revoked by Bot"}
        resp = requests.post("https://graph.microsoft.com/v1.0/roleManagement/directory/roleAssignmentScheduleRequests", json=payload, headers=headers)
        if resp.status_code in [200, 201]: return f"✅ {role_name} 회수 완료"
        return f"❌ 오류: {resp.json().get('error', {}).get('message')}"
    except Exception as e: return f"❌ 에러: {e}"

async def assign_directory_role_pim(email, role_name, duration="P30D"):
    try:
        token = get_access_token()
        uid = await _get_user_id(email, token)
        if not uid: return f"❌ 사용자 못 찾음: {email}"
        role_id = await _get_role_definition_id(role_name, token)
        if not role_id: return f"❌ 역할 이름 오류: '{role_name}'"
        headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
        payload = {"action": "adminAssign", "principalId": uid, "roleDefinitionId": role_id, "directoryScopeId": "/", "scheduleInfo": {"expiration": {"type": "afterDuration", "duration": duration}}, "justification": "Assigned by Bot"}
        resp = requests.post("https://graph.microsoft.com/v1.0/roleManagement/directory/roleAssignmentScheduleRequests", json=payload, headers=headers)
        if resp.status_code in [200, 201]: return f"👮‍♂️ {role_name} 부여 완료"
        return f"❌ 오류: {resp.json().get('error', {}).get('message')}"
    except Exception as e: return f"❌ 에러: {e}"

# ---------------------------------------------------------
# [기능 2: 라이선스 관리]
# ---------------------------------------------------------
async def remove_license(email, keyword):
    try:
        token = get_access_token()
        uid = await _get_user_id(email, token)
        if not uid: return f"❌ 사용자 못 찾음: {email}"
        headers = {"Authorization": f"Bearer {token}"}
        skus = requests.get("https://graph.microsoft.com/v1.0/subscribedSkus", headers=headers).json().get('value', [])
        target = _find_best_sku_match(keyword, skus)
        if not target: return f"❌ 라이선스 '{keyword}' 못 찾음"
        payload = {"addLicenses": [], "removeLicenses": [target['skuId']]}
        resp = requests.post(f"https://graph.microsoft.com/v1.0/users/{uid}/assignLicense", json=payload, headers=headers)
        if resp.status_code == 200: return f"🗑️ {target['skuPartNumber']} 회수됨"
        return f"❌ 실패: {resp.text}"
    except Exception as e: return f"❌ 에러: {e}"

async def assign_license(email, keyword):
    try:
        token = get_access_token()
        uid = await _get_user_id(email, token)
        if not uid: return f"❌ 사용자 못 찾음: {email}"
        headers = {"Authorization": f"Bearer {token}"}
        skus = requests.get("https://graph.microsoft.com/v1.0/subscribedSkus", headers=headers).json().get('value', [])
        target = _find_best_sku_match(keyword, skus)
        if not target: return f"❌ 라이선스 '{keyword}' 못 찾음"
        payload = {"addLicenses": [{"disabledPlans": [], "skuId": target['skuId']}], "removeLicenses": []}
        resp = requests.post(f"https://graph.microsoft.com/v1.0/users/{uid}/assignLicense", json=payload, headers=headers)
        if resp.status_code == 200: return f"✅ {target['skuPartNumber']} 할당됨"
        return f"❌ 실패: {resp.text}"
    except Exception as e: return f"❌ 에러: {e}"

# ---------------------------------------------------------
# [기능 3: 사용자 생성 및 조회]
# ---------------------------------------------------------
async def create_user(upn, display_name, department, job_title, password="InitialPassword123!"):
    try:
        token = get_access_token()
        headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
        payload = {
            "accountEnabled": True, "displayName": display_name, "mailNickname": upn.split('@')[0],
            "userPrincipalName": upn, "department": department, "jobTitle": job_title,
            "passwordProfile": {"forceChangePasswordNextSignIn": True, "password": password}
        }
        resp = requests.post("https://graph.microsoft.com/v1.0/users", json=payload, headers=headers)
        if resp.status_code == 201: return True, f"✅ 계정 생성: {upn}"
        return False, f"❌ 생성 실패: {resp.json().get('error', {}).get('message')}"
    except Exception as e: return False, f"❌ 에러: {e}"

async def check_user_exists(upn):
    try:
        token = get_access_token()
        resp = requests.get(f"https://graph.microsoft.com/v1.0/users/{upn}", headers={"Authorization": f"Bearer {token}"})
        return resp.status_code == 200
    except: return False

async def get_tenant_license_names():
    try:
        token = get_access_token()
        headers = {"Authorization": f"Bearer {token}"}
        resp = requests.get("https://graph.microsoft.com/v1.0/subscribedSkus", headers=headers)
        if resp.status_code == 200:
            return [item['skuPartNumber'] for item in resp.json().get('value', [])]
        return []
    except: return []

async def get_user_admin_roles(user_aad_id):
    if not user_aad_id: return []
    try:
        token = get_access_token()
        headers = {"Authorization": f"Bearer {token}"}
        resp = requests.get(f"https://graph.microsoft.com/v1.0/users/{user_aad_id}/memberOf", headers=headers)
        if resp.status_code == 200:
            return [i['displayName'] for i in resp.json().get('value', []) if i.get('@odata.type') == "#microsoft.graph.directoryRole"]
        return []
    except: return []

# ---------------------------------------------------------
# [기능 4: Azure 구독 및 비용 조회]
# ---------------------------------------------------------
async def get_owner_subscriptions(user_aad_id):
    if not user_aad_id: return []
    try:
        arm_token = get_arm_token()
        graph_token = get_access_token()
        if not arm_token or not graph_token: return []
        
        arm_headers = {"Authorization": f"Bearer {arm_token}"}
        graph_headers = {"Authorization": f"Bearer {graph_token}"}

        # 사용자 UPN 조회 (클래식 관리자 확인용)
        user_upn = None
        u_resp = requests.get(f"https://graph.microsoft.com/v1.0/users/{user_aad_id}?$select=userPrincipalName,mail", headers=graph_headers)
        if u_resp.status_code == 200:
            d = u_resp.json()
            user_upn = d.get('userPrincipalName') or d.get('mail')

        resp = requests.get("https://management.azure.com/subscriptions?api-version=2020-01-01", headers=arm_headers)
        if resp.status_code != 200: return []
        
        all_subs = resp.json().get('value', [])
        valid_subs = []
        owner_role_guid = "8e3af657-a8ff-443c-a75d-2fe8c4bcb635"

        for sub in all_subs:
            sub_id = sub['subscriptionId']
            is_owner = False
            
            # Check 1: RBAC Owner
            rbac_resp = requests.get(f"https://management.azure.com/subscriptions/{sub_id}/providers/Microsoft.Authorization/roleAssignments?api-version=2015-07-01&$filter=principalId eq '{user_aad_id}'", headers=arm_headers)
            if rbac_resp.status_code == 200:
                for assignment in rbac_resp.json().get('value', []):
                    if owner_role_guid in assignment.get('properties', {}).get('roleDefinitionId', ''):
                        is_owner = True; break
            
            # Check 2: Classic Co-Administrator
            if not is_owner and user_upn:
                classic_resp = requests.get(f"https://management.azure.com/subscriptions/{sub_id}/providers/Microsoft.Authorization/classicAdministrators?api-version=2015-07-01", headers=arm_headers)
                if classic_resp.status_code == 200:
                    for admin in classic_resp.json().get('value', []):
                        if admin.get('properties', {}).get('emailAddress', '').lower() == user_upn.lower():
                            is_owner = True; break
            
            if is_owner: valid_subs.append({"id": sub_id, "name": sub['displayName']})
        return valid_subs
    except: return []

# [NEW] 상세 비용 분석 함수 (Power BI 스타일 데이터 수집)
async def get_advanced_cost_data(subscription_id, timeframe="WeekToDate"):
    if not subscription_id: return None
    try:
        token = get_arm_token()
        headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
        base_url = f"https://management.azure.com/subscriptions/{subscription_id}/providers/Microsoft.CostManagement/query?api-version=2021-10-01"

        def _get_query_payload(group_by_field):
            return {
                "type": "ActualCost",
                "dataSet": {
                    "granularity": "None",
                    "aggregation": {"totalCost": {"name": "Cost", "function": "Sum"}},
                    "grouping": [{"type": "Dimension", "name": group_by_field}]
                },
                "timeframe": timeframe
            }

        # 1. Resource ID별 조회 (개별 VM/리소스 단위 확인)
        resp_res = requests.post(base_url, json=_get_query_payload("ResourceId"), headers=headers)
        top_resources = []
        total_cost = 0.0
        
        if resp_res.status_code == 200:
            rows = resp_res.json().get('properties', {}).get('rows', [])
            raw_data = []
            for r in rows:
                cost = float(r[0])
                full_id = r[1]
                # 리소스 이름만 추출 (예: .../virtualMachines/my-vm-01 -> my-vm-01)
                res_name = full_id.split('/')[-1]
                res_type = full_id.split('/')[-2] if len(full_id.split('/')) > 2 else "Resource"
                
                raw_data.append({"name": res_name, "type": res_type, "cost": cost})
                total_cost += cost
            
            # 비용 내림차순 정렬 후 Top 10 추출
            raw_data.sort(key=lambda x: x['cost'], reverse=True)
            top_resources = raw_data[:10]
        
        # 2. Location(지역)별 조회
        resp_loc = requests.post(base_url, json=_get_query_payload("ResourceLocation"), headers=headers)
        by_location = []
        if resp_loc.status_code == 200:
            rows = resp_loc.json().get('properties', {}).get('rows', [])
            for r in rows:
                by_location.append({"name": r[1], "cost": float(r[0])})
            by_location.sort(key=lambda x: x['cost'], reverse=True)

        # 3. Resource Group(리소스 그룹)별 조회
        resp_rg = requests.post(base_url, json=_get_query_payload("ResourceGroupName"), headers=headers)
        by_rg = []
        if resp_rg.status_code == 200:
            rows = resp_rg.json().get('properties', {}).get('rows', [])
            for r in rows:
                by_rg.append({"name": r[1], "cost": float(r[0])})
            by_rg.sort(key=lambda x: x['cost'], reverse=True)

        return {
            "total": total_cost,
            "top_10_resources": top_resources, # 개별 리소스 Top 10
            "by_location": by_location,        # 지역별 비용
            "by_rg": by_rg                     # 리소스 그룹별 비용
        }

    except Exception as e:
        print(f"❌ Advanced Cost API Error: {e}")
        return None
import os
import json
from openai import AsyncAzureOpenAI
from azure.core.credentials import AzureKeyCredential
from azure.search.documents import SearchClient
from dotenv import load_dotenv
import graph_helper # Graph 정보 연동

load_dotenv()

# [AI Search 설정]
SEARCH_ENDPOINT = os.getenv("AZURE_SEARCH_ENDPOINT", "")
SEARCH_KEY = os.getenv("AZURE_SEARCH_KEY", "")
SEARCH_INDEX = os.getenv("AZURE_SEARCH_INDEX", "")
 
async def search_company_knowledge(query):
    print(f"\n🔍 [AI Search 시작] 검색어: {query}", flush=True)
    try:
        if not SEARCH_ENDPOINT or not SEARCH_KEY or not SEARCH_INDEX:
            return None
        credential = AzureKeyCredential(SEARCH_KEY)
        client = SearchClient(endpoint=SEARCH_ENDPOINT, index_name=SEARCH_INDEX, credential=credential)
        results = list(client.search(search_text=query, top=3))
        if not results: return None
        docs = []
        for i, r in enumerate(results):
            doc_body = (r.get('content') or r.get('merged_content') or r.get('chunk') or r.get('text') or r.get('snippet') or "")
            if doc_body: docs.append(f"- {doc_body}")
        return "\n".join(docs) if docs else None
    except Exception as e:
        print(f"❌ [AI Search 에러] {e}", flush=True)
        return None

async def get_chat_decision(user_query, commander_roles):
    client = AsyncAzureOpenAI(
        azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
        api_key=os.getenv("AZURE_OPENAI_API_KEY"),
        api_version="2024-02-15-preview"
    )

    # 1. 사내 지식 검색
    company_info = await search_company_knowledge(user_query)
    company_context = f"사내 문서 내용:\n{company_info}" if company_info else "사내 문서 없음. MS 공식 문서 기준 답변."

    # 2. [핵심] 현재 테넌트의 실제 보유 라이선스 조회
    valid_licenses = await graph_helper.get_tenant_license_names()
    license_context = f"현재 테넌트 보유 라이선스(SKU) 목록: {valid_licenses}"

    # 3. 시스템 프롬프트 구성 (한글 -> 영문 자동 변환 강화)
    system_prompt = f"""
    당신은 기업용 보안 및 IT 지원 봇입니다.
    현재 사용자 권한: {commander_roles}
    
    [컨텍스트]
    {company_context}
    {license_context}

    [역할 및 라이선스 매칭 규칙 - 매우 중요]
    1. **라이선스 매칭:** - 사용자가 "E3", "오피스" 등으로 말하면, 위 '보유 라이선스 목록' 중 가장 유사한 정확한 SKU Name(예: SPE_E3, O365_BUSINESS_PREMIUM)으로 변환해서 'detail' 필드에 넣으세요.
    
    2. **관리자 권한 매칭 (한글/영어 자동 변환):**
       - 사용자가 **한글**로 관리자 역할을 말하더라도, 반드시 **Microsoft Entra ID 공식 영문명**으로 번역해서 'detail' 필드에 넣으세요.
       - **매칭 예시:**
         - "전역 관리자" -> "Global Administrator"
         - "사용자 관리자" -> "User Administrator"
         - "패브릭 관리자" -> "Fabric Administrator"
         - "청구 관리자" -> "Billing Administrator"
         - "권한 있는 역할 관리자" -> "Privileged Role Administrator"
       - **절대 한글 이름을 그대로 'detail'에 넣지 마세요.** (Graph API는 영문 이름만 인식합니다.)

    [행동 규칙]
    1. SECURITY_ALERT: 비밀번호/계정 요구 시.
    2. WORK: 권한/라이선스 부여(ASSIGN) 또는 회수(REMOVE).
       - intent: 'ADMIN_ROLE' 또는 'LICENSE'
       - detail: 위 매칭 규칙에 따라 변환된 정확한 값 (SKU명 또는 Role 영문명)
    3. EXCEL_GUIDE: 대량 작업 요청 시.
    4. CHAT: 일반 질문.

    반드시 JSON 형식으로 응답:
    {{
        "type": "WORK" | "CHAT" | "SECURITY_ALERT" | "AUDIT_QUERY" | "EXCEL_GUIDE",
        "decision": "APPROVE" | "REJECT",
        "action": "ASSIGN" | "REMOVE" | null,
        "intent": "ADMIN_ROLE" | "LICENSE" | null,
        "target_email": "target@domain.com" | null,
        "detail": "SKU_NAME_OR_ROLE_NAME_IN_ENGLISH",
        "content": "답변"
    }}
    """

    response = await client.chat.completions.create(
        model=os.getenv("AZURE_OPENAI_DEPLOYMENT_NAME"),
        messages=[
            {"role": "system", "content": system_prompt}, 
            {"role": "user", "content": user_query}
        ],
        response_format={ "type": "json_object" } 
    )
    
    try:
        return json.loads(response.choices[0].message.content)
    except:
        return {"type": "CHAT", "content": "시스템 오류"}
import pandas as pd
import io
import os
import matplotlib
import matplotlib.pyplot as plt
from reportlab.lib.pagesizes import letter
from reportlab.pdfgen import canvas
from reportlab.lib.utils import ImageReader
from pptx import Presentation
from pptx.util import Inches, Pt
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont

# 서버 환경(GUI 없음)용 설정
matplotlib.use('Agg')
 
def register_font():
    font_path = "NanumGothic.ttf" 
    if os.path.exists(font_path):
        try:
            pdfmetrics.registerFont(TTFont("KoreanFont", font_path))
            return "KoreanFont"
        except:
            return "Helvetica"
    return "Helvetica"

# [기존 기능 유지] 엑셀 처리
def parse_excel_and_generate_pdf(file_content):
    try:
        df = pd.read_excel(io.BytesIO(file_content), engine='openpyxl')
        if df.shape[1] < 2: return None
        cols = df.columns
        summary = df.groupby(cols[0])[cols[1]].sum().to_dict()
        buffer = io.BytesIO()
        c = canvas.Canvas(buffer, pagesize=letter)
        font_name = register_font()
        c.setFont(font_name, 16)
        c.drawString(100, 750, "Weekly Report")
        c.line(100, 740, 500, 740)
        c.setFont(font_name, 12)
        y = 700
        total = 0.0
        for k, v in summary.items():
            try:
                val = float(v)
                line_text = f"{k}: {val:,.2f}"
                total += val
            except:
                line_text = f"{k}: {v}"
            c.drawString(100, y, line_text)
            y -= 25
        c.line(100, y+5, 300, y+5)
        c.drawString(100, y-20, f"Total: {total:,.2f}")
        c.save()
        buffer.seek(0)
        return buffer.getvalue()
    except Exception as e:
        print(f"PDF Error: {e}")
        return None

# [NEW] 가로 바 차트 (Top 10 리소스용)
def create_bar_chart(data, title="Top 10 Resources"):
    if not data: return None
    names = [d['name'][:15] for d in data[:10]] # 이름 너무 길면 자름
    values = [d['cost'] for d in data[:10]]
    
    plt.rcParams['font.family'] = 'NanumGothic'
    plt.rcParams['axes.unicode_minus'] = False
    
    fig, ax = plt.subplots(figsize=(6, 4))
    
    # 가로 바 차트 (비용 높은 순서대로 위쪽 배치)
    y_pos = range(len(names))
    ax.barh(y_pos, values, align='center', color='skyblue')
    ax.set_yticks(y_pos)
    ax.set_yticklabels(names)
    ax.invert_yaxis() 
    ax.set_xlabel('Cost (KRW)')
    ax.set_title(title)
    
    img_buf = io.BytesIO()
    plt.savefig(img_buf, format='png', bbox_inches='tight')
    plt.close(fig)
    img_buf.seek(0)
    return img_buf

# [NEW] 파이 차트 (지역/리소스그룹용)
def create_pie_chart(data, title="Breakdown"):
    if not data: return None
    labels = [d['name'] for d in data[:5]] 
    sizes = [d['cost'] for d in data[:5]]
    
    plt.rcParams['font.family'] = 'NanumGothic'
    plt.rcParams['axes.unicode_minus'] = False
    
    fig, ax = plt.subplots(figsize=(4, 3))
    ax.pie(sizes, labels=labels, autopct='%1.1f%%', startangle=90)
    ax.axis('equal') 
    plt.title(title, fontsize=10)
    
    img_buf = io.BytesIO()
    plt.savefig(img_buf, format='png', bbox_inches='tight')
    plt.close(fig)
    img_buf.seek(0)
    return img_buf

# [NEW] PDF 리포트 (Power BI Style)
def generate_cost_pdf(cost_data_list):
    buffer = io.BytesIO()
    c = canvas.Canvas(buffer, pagesize=letter)
    font_name = register_font()
    
    y = 750
    c.setFont(font_name, 20)
    c.drawString(50, y, "Weekly Azure Cost Report (Power BI Style)")
    y -= 40
    
    for sub in cost_data_list:
        if y < 400: 
            c.showPage()
            y = 750
        
        # 구독 헤더
        c.setFont(font_name, 16)
        c.drawString(50, y, f"Subscription: {sub['sub_name']}")
        c.setFont(font_name, 14)
        c.drawString(50, y-20, f"Total: ₩{sub['total']:,.0f}")
        y -= 40
        
        # 1. Top 10 Resources Chart (왼쪽 배치)
        res_chart = create_bar_chart(sub['top_10_resources'], title="Top 10 Expensive Resources")
        if res_chart:
            img = ImageReader(res_chart)
            c.drawImage(img, 50, y-220, width=300, height=200)
        
        # 2. Location Chart (오른쪽 위)
        loc_chart = create_pie_chart(sub['by_location'], title="Cost by Region")
        if loc_chart:
            img = ImageReader(loc_chart)
            c.drawImage(img, 360, y-120, width=200, height=150)
            
        # 3. Resource Group Chart (오른쪽 아래)
        rg_chart = create_pie_chart(sub['by_rg'], title="Cost by Resource Group")
        if rg_chart:
            img = ImageReader(rg_chart)
            c.drawImage(img, 360, y-250, width=200, height=150)

        y -= 260
        
        # 텍스트 상세 리스트 (Top 5만 텍스트로 표기)
        c.setFont(font_name, 10)
        c.drawString(50, y, "🏆 Top 5 Detailed Resources:")
        ty = y - 15
        for item in sub['top_10_resources'][:5]:
            # 리소스 타입과 이름 함께 표기
            text = f"- [{item['type']}] {item['name']}: ₩{item['cost']:,.0f}"
            c.drawString(60, ty, text)
            ty -= 12
        y = ty - 40

    c.save()
    buffer.seek(0)
    return buffer.getvalue()

# [NEW] PPT 리포트 (Power BI Style)
def generate_cost_ppt(cost_data_list):
    prs = Presentation()
    
    # 타이틀 슬라이드
    title_layout = prs.slide_layouts[0]
    slide = prs.slides.add_slide(title_layout)
    slide.shapes.title.text = "Weekly Azure Cost Analysis"
    slide.placeholders[1].text = "Granular Resource & Region Breakdown"
    
    for sub in cost_data_list:
        # 내용 슬라이드 (빈 레이아웃 사용 후 이미지 배치)
        blank_layout = prs.slide_layouts[6] 
        slide = prs.slides.add_slide(blank_layout)
        
        # 제목
        txBox = slide.shapes.add_textbox(Inches(0.5), Inches(0.2), Inches(9), Inches(0.8))
        tf = txBox.text_frame
        tf.text = f"{sub['sub_name']} (Total: ₩{sub['total']:,.0f})"
        tf.paragraphs[0].font.size = Pt(24)
        tf.paragraphs[0].font.bold = True
        
        # 1. Top 10 Chart (왼쪽)
        res_chart = create_bar_chart(sub['top_10_resources'])
        if res_chart:
            slide.shapes.add_picture(res_chart, Inches(0.5), Inches(1.5), width=Inches(4.5))

        # 2. Location Chart (오른쪽 상단)
        loc_chart = create_pie_chart(sub['by_location'], title="By Region")
        if loc_chart:
            slide.shapes.add_picture(loc_chart, Inches(5.5), Inches(1.5), width=Inches(3.5))

        # 3. Resource Group Chart (오른쪽 하단)
        rg_chart = create_pie_chart(sub['by_rg'], title="By Resource Group")
        if rg_chart:
            slide.shapes.add_picture(rg_chart, Inches(5.5), Inches(4.5), width=Inches(3.5))
            
    buffer = io.BytesIO()
    prs.save(buffer)
    buffer.seek(0)
    return buffer.getvalue()
aiohttp==3.9.3
botbuilder-core==4.15.0
botbuilder-schema==4.15.0
botbuilder-integration-aiohttp==4.15.0
azure-identity==1.15.0
azure-search-documents==11.4.0
requests==2.31.0
pandas==2.2.0
tabulate==0.9.0
xlsxwriter==3.1.9
apscheduler==3.10.4
openai==1.12.0
pyfiglet==0.8.post1
python-dotenv==1.0.0
httpx==0.27.0
openpyxl==3.1.2
fpdf==1.7.2
python-pptx==0.6.23
matplotlib==3.8.2
reportlab==4.0.0
azure-storage-blob==12.19.0
 
