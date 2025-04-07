import asyncio
import os
import json
from datetime import datetime
import pytz
import aiohttp
from nacl.signing import SigningKey
from base58 import b58encode, b58decode
from colorama import Fore, Style, init

# 针对 Windows 设置 SelectorEventLoop
if os.name == 'nt':
    asyncio.set_event_loop_policy(asyncio.WindowsSelectorEventLoopPolicy())

init(autoreset=True)
CST = pytz.timezone('Asia/Shanghai')

class Flow3Master:
    def __init__(self):
        self.accounts = {}
        self.proxies = []
        self.proxy_index = 0 
        self.session = None
        self.base_url = "https://api.flow3.tech/api/v1"
        self.default_referral_code = "2XW5GfHbt"  # 默认推荐码，可能需要更新为有效值
        self.semaphore = asyncio.Semaphore(5)  # 限制最多 5 个并发请求

    def print(self, msg, tag="日志", color=Fore.WHITE):
        time = datetime.now(CST).strftime("%H:%M:%S")
        tag_styles = {
            "日志": f"{Fore.CYAN}📜{Style.RESET_ALL}",
            "成功": f"{Fore.GREEN}✅{Style.RESET_ALL}",
            "错误": f"{Fore.RED}❌{Style.RESET_ALL}",
            "警告": f"{Fore.YELLOW}⚠️{Style.RESET_ALL}",
            "任务": f"{Fore.MAGENTA}📋{Style.RESET_ALL}"
        }
        print(f"{Fore.BLUE}[{time}]{Style.RESET_ALL} {tag_styles.get(tag, tag)} {color}{msg}{Style.RESET_ALL}")

    def banner(self):
        os.system('cls' if os.name == 'nt' else 'clear')
        print(f"{Fore.CYAN}{'✨' * 40}{Style.RESET_ALL}")
        print(f"{Fore.GREEN}Flow3Master - 智能自动化助手{Style.RESET_ALL}".center(80))
        print(f"{Fore.YELLOW}关注X：https://x.com/qklxsqf | 使用邀请链接：https://dashboard.flow3.tech?ref=2XW5GfHbt{Style.RESET_ALL}".center(80))
        print(f"{Fore.CYAN}{'✨' * 40}{Style.RESET_ALL}")

    async def load_config(self):
        if not os.path.exists("accounts.txt"):
            self.print("未找到 accounts.txt，请创建并添加 Solana 私钥", "错误", Fore.RED)
            return False
        with open("accounts.txt", "r", encoding="utf-8") as f:
            for line in f:
                line = line.strip()
                if line:
                    parts = line.split(":")
                    pk = parts[0]
                    referral_code = parts[1] if len(parts) > 1 else self.default_referral_code
                    address = self.get_address(pk)
                    if address:
                        self.accounts[address] = {"private_key": pk, "token": None, "proxy": None, "referral_code": referral_code}
                    else:
                        self.print(f"无效私钥: {pk[:6]}...", "错误", Fore.RED)
        self.print(f"加载了 {len(self.accounts)} 个账户", "成功", Fore.GREEN)

        if os.path.exists("proxies.txt"):
            with open("proxies.txt", "r", encoding="utf-8") as f:
                self.proxies = [line.strip() for line in f if line.strip()]
            self.print(f"加载了 {len(self.proxies)} 个代理", "成功", Fore.GREEN)
            for addr in self.accounts:
                self.accounts[addr]["proxy"] = self.get_next_proxy()
        else:
            self.print("未找到 proxies.txt，将使用默认 IP", "警告", Fore.YELLOW)
        return True

    def get_next_proxy(self):
        if not self.proxies:
            return None
        proxy = self.proxies[self.proxy_index]
        self.proxy_index = (self.proxy_index + 1) % len(self.proxies)
        return f"http://{proxy}" if not proxy.startswith("http") else proxy

    def get_address(self, private_key):
        try:
            key_bytes = b58decode(private_key)[:32]
            signing_key = SigningKey(key_bytes)
            return b58encode(signing_key.verify_key.encode()).decode()
        except Exception as e:
            self.print(f"解析私钥失败: {e}", "错误", Fore.RED)
            return None

    def sign_message(self, private_key, referral_code):
        msg = "Please sign this message to connect your wallet to Flow 3 and verifying your ownership only."
        try:
            key_bytes = b58decode(private_key)[:32]
            signing_key = SigningKey(key_bytes)
            signature = signing_key.sign(msg.encode("utf-8"))
            return {
                "message": msg,
                "walletAddress": self.get_address(private_key),
                "signature": b58encode(signature.signature).decode(),
                "referralCode": referral_code
            }
        except Exception as e:
            self.print(f"生成签名失败: {e}", "错误", Fore.RED)
            return None

    async def request(self, method, endpoint, address=None, data=None, proxy=None, retries=5):
        url = f"{self.base_url}{endpoint}"
        headers = {"Content-Type": "application/json"}
        if address and self.accounts[address]["token"]:
            headers["Authorization"] = f"Bearer {self.accounts[address]['token']}"
        if method == "POST" and data is None:
            headers["Content-Length"] = "0"
        
        async with self.semaphore:  # 使用信号量限制并发
            for attempt in range(retries):
                try:
                    async with self.session.request(
                        method, url, headers=headers, json=data, proxy=proxy, timeout=aiohttp.ClientTimeout(total=60)
                    ) as resp:
                        resp_text = await resp.text()
                        if resp.status == 401:
                            self.print(f"账户 {self.mask(address)} Token 失效，重新登录", "警告", Fore.YELLOW)
                            await self.authenticate(address, self.accounts[address]["private_key"], proxy)
                            headers["Authorization"] = f"Bearer {self.accounts[address]['token']}"
                            continue
                        elif resp.status == 429:  # 处理 Too Many Requests
                            self.print(f"请求 {url} 返回 429: 速率限制，等待重试", "警告", Fore.YELLOW)
                            await asyncio.sleep(10)  # 遇到 429 时等待 10 秒
                            continue
                        elif resp.status == 400:
                            self.print(f"请求 {url} 返回 400: {resp_text}", "错误", Fore.RED)
                            return await resp.json()
                        resp.raise_for_status()
                        result = await resp.json()
                        if result is None:
                            self.print(f"请求 {url} 返回空响应", "警告", Fore.YELLOW)
                        return result
                except aiohttp.ClientResponseError as e:
                    error_msg = f"{e.status}, message='{e.message}'"
                    if e.status == 429 and attempt < retries - 1:
                        self.print(f"请求 {url} 失败，重试 {attempt + 1}/{retries}: {error_msg}", "警告", Fore.YELLOW)
                        await asyncio.sleep(10)  # 遇到 429 时等待更长时间
                    elif attempt < retries - 1:
                        self.print(f"请求 {url} 失败，重试 {attempt + 1}/{retries}: {error_msg}", "警告", Fore.YELLOW)
                        await asyncio.sleep(3)
                    else:
                        self.print(f"请求 {url} 失败: {error_msg}", "错误", Fore.RED)
                        if proxy and self.proxies:
                            new_proxy = self.get_next_proxy()
                            self.accounts[address]["proxy"] = new_proxy
                            self.print(f"切换代理为: {new_proxy}", "日志")
                        return None
                except Exception as e:
                    error_msg = str(e)
                    if attempt < retries - 1:
                        self.print(f"请求 {url} 失败，重试 {attempt + 1}/{retries}: {error_msg}", "警告", Fore.YELLOW)
                        await asyncio.sleep(3)
                    else:
                        self.print(f"请求 {url} 失败: {error_msg}", "错误", Fore.RED)
                        if proxy and self.proxies:
                            new_proxy = self.get_next_proxy()
                            self.accounts[address]["proxy"] = new_proxy
                            self.print(f"切换代理为: {new_proxy}", "日志")
                        return None

    async def authenticate(self, address, private_key, proxy):
        self.print(f"账户 {self.mask(address)} 正在登录...", "日志")
        referral_code = self.accounts[address].get("referral_code", self.default_referral_code)
        payload = self.sign_message(private_key, referral_code)
        if not payload:
            return False
        resp = await self.request("POST", "/user/login", data=payload, proxy=proxy)
        if resp and isinstance(resp, dict) and "data" in resp and "accessToken" in resp["data"]:
            self.accounts[address]["token"] = resp["data"]["accessToken"]
            self.print(f"账户 {self.mask(address)} 登录成功 (推荐码: {referral_code})", "成功", Fore.GREEN)
            return True
        self.print(f"账户 {self.mask(address)} 登录失败: {resp}", "错误", Fore.RED)
        return False

    async def ping(self, address, private_key, proxy):
        if not self.accounts[address]["token"]:
            self.print(f"账户 {self.mask(address)} 未登录，跳过带宽分享", "警告", Fore.YELLOW)
            return False
        resp = await self.request("POST", "/bandwidth", address, proxy=proxy)
        if resp and isinstance(resp, dict) and "data" in resp:
            self.print(f"账户 {self.mask(address)} 带宽分享成功", "成功", Fore.GREEN)
            return True
        self.print(f"账户 {self.mask(address)} 带宽分享失败: {resp}", "错误", Fore.RED)
        return False

    async def fetch_stats(self, address, private_key, proxy):
        if not self.accounts[address]["token"]:
            self.print(f"账户 {self.mask(address)} 未登录，跳过统计获取", "警告", Fore.YELLOW)
            return None
        resp = await self.request("GET", "/tasks/stats", address, proxy=proxy)
        if resp and isinstance(resp, dict) and "data" in resp:
            data = resp["data"]
            self.print(
                f"账户 {self.mask(address)} 统计 - "
                f"带宽: {data.get('totalBandwidthReward', 0)} PTS | "
                f"总计: {data.get('totalRewardPoint', 0)} PTS | "
                f"邀请: {data.get('totalReferralRewardPoint', 0)} PTS | "
                f"任务: {data.get('totalTaskRewardPoint', 0)} PTS",
                "成功", Fore.GREEN
            )
            return data
        self.print(f"账户 {self.mask(address)} 获取统计失败: {resp}", "错误", Fore.RED)
        return None

    async def checkin(self, address, private_key, proxy):
        if not self.accounts[address]["token"]:
            self.print(f"账户 {self.mask(address)} 未登录，跳过签到", "警告", Fore.YELLOW)
            return False
        resp = await self.request("POST", "/tasks/complete-daily", address, proxy=proxy)
        if resp and isinstance(resp, dict):
            if resp.get("message") == "Complete daily tasks successfully":
                self.print(f"账户 {self.mask(address)} 签到成功", "成功", Fore.GREEN)
                return True
            elif resp.get("statusCode") == 400 or "bad request" in resp.get("message", "").lower():
                self.print(f"账户 {self.mask(address)} 今日已签到", "警告", Fore.YELLOW)
                return True
            else:
                self.print(f"账户 {self.mask(address)} 签到失败: {resp.get('message', '未知错误')}", "错误", Fore.RED)
                return False
        self.print(f"账户 {self.mask(address)} 签到失败: 无响应或无效数据 {resp}", "错误", Fore.RED)
        return False

    async def manage_tasks(self, address, private_key, proxy):
        if not self.accounts[address]["token"]:
            self.print(f"账户 {self.mask(address)} 未登录，跳过任务管理", "警告", Fore.YELLOW)
            return False
        resp = await self.request("GET", "/tasks/", address, proxy=proxy)
        if resp and isinstance(resp, dict) and "data" in resp:
            tasks = resp["data"]
            if not isinstance(tasks, list):
                self.print(f"账户 {self.mask(address)} 任务列表无效: {tasks}", "错误", Fore.RED)
                return False
            for task in tasks:
                if task.get("status") == 0:
                    task_id = task.get("taskId")
                    title = task.get("title", "未知任务")
                    self.print(f"账户 {self.mask(address)} 执行任务: {title}", "任务", Fore.MAGENTA)
                    task_resp = await self.request("POST", f"/tasks/{task_id}/complete", address, proxy=proxy)
                    if task_resp and isinstance(task_resp, dict) and task_resp.get("message") == "Complete tasks successfully":
                        self.print(f"账户 {self.mask(address)} 任务 {title} 完成", "成功", Fore.GREEN)
            return True
        self.print(f"账户 {self.mask(address)} 获取任务失败: {resp}", "错误", Fore.RED)
        return False

    def mask(self, address):
        return f"{address[:6]}...{address[-6:]}"

    async def run_ping(self, address, private_key, proxy):
        while True:
            await self.ping(address, private_key, proxy)
            await asyncio.sleep(60)

    async def run_stats(self, address, private_key, proxy):
        while True:
            await self.fetch_stats(address, private_key, proxy)
            await asyncio.sleep(600)

    async def run_checkin(self, address, private_key, proxy):
        while True:
            await self.checkin(address, private_key, proxy)
            await asyncio.sleep(12 * 3600)

    async def run_tasks(self, address, private_key, proxy):
        while True:
            await self.manage_tasks(address, private_key, proxy)
            await asyncio.sleep(24 * 3600)

    async def run_account(self, address, private_key):
        proxy = self.accounts[address]["proxy"]
        proxy_str = proxy or "无代理"
        self.print(f"启动账户 {self.mask(address)} | 代理: {proxy_str}", "日志")

        if not self.accounts[address]["token"]:
            success = await self.authenticate(address, private_key, proxy)
            if not success:
                self.print(f"账户 {self.mask(address)} 登录失败，跳过后续任务", "错误", Fore.RED)
                return

        tasks = [
            self.run_ping(address, private_key, proxy),
            self.run_stats(address, private_key, proxy),
            self.run_checkin(address, private_key, proxy),
            self.run_tasks(address, private_key, proxy)
        ]
        await asyncio.gather(*tasks)

    async def main(self):
        self.banner()
        if not await self.load_config():
            return

        async with aiohttp.ClientSession() as session:
            self.session = session
            tasks = [self.run_account(addr, data["private_key"]) for addr, data in self.accounts.items()]
            await asyncio.gather(*tasks)

if __name__ == "__main__":
    try:
        bot = Flow3Master()
        asyncio.run(bot.main())
    except KeyboardInterrupt:
        print(f"{Fore.RED}程序已终止，感谢使用 Flow3Master!{Style.RESET_ALL}")
    except Exception as e:
        print(f"{Fore.RED}发生错误: {e}{Style.RESET_ALL}")
