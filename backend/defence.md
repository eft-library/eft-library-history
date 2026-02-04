# 📂 목록

- [다국어 지원](./i18n_data.md)
- [ORM 사용 (With SqlAlchemy)](./orm.md)
- [Google 로그인 도입기: 사용자 피드백으로 시작된 변화](./token_check.md)
- [사이트 공격 대비](./defence.md)

# 사이트 공격 대비

요즘 백엔드 API 요청 로그를 확인하면, 꾸준하게 env와 같은 정보들을 가져가려는 움직임이 많이 보입니다.  
지금도 404로 가서 문제 없지만, 그래도 요청을 막는게 좋을 것 같아 로직을 추가했습니다.

**FastAPI의 env 정보를 가져가려는 모습**
<img width="618" height="22" alt="스크린샷 2026-02-04 오전 9 59 23" src="https://github.com/user-attachments/assets/3c6fe83d-f578-487e-9466-23ab9d1e04ea" />

**Wordpress인 줄 알고 정보를 가져가려는 모습**
<img width="926" height="357" alt="스크린샷 2026-02-04 오전 9 59 44" src="https://github.com/user-attachments/assets/b840a5ed-63c4-4ea8-b614-97685fe1eaab" />

**CMS / PHP / JSP 취약 파일 스캔**
<img width="1202" height="250" alt="스크린샷 2026-02-04 오전 10 07 33" src="https://github.com/user-attachments/assets/c67ec2d0-a8f2-4a82-8d04-242c95bbb411" />

## .env 요청

다행히도 .env 요청은 지금처럼 404를 보내면 된다고 Claude랑 GPT가 알려줘서 현상 유지 하기로 했습니다.  
요청이 왔을때 억지로 답변을 만들어서 보내는 것이 더 안좋다고 합니다.

## Wordpress인 줄 알고 정보를 가져가려는 모습

이것 또한 .env 요청 처럼 404를 보내는 것으로 현상 유지 하기로 했습니다.

## API 요청 필터링 하기

CMS / PHP / JSP 취약 파일 스캔 요청에 대해서 필터링을 먼저 적용했습니다.

1. 도메인에 맞지 않는 확장자가 붙은 요청들은 전부 정규식을 사용하여 필터링 해줍니다.
2. 사이트에 파일 업로드 기능이 있는데 해당 내용을 제외하고 다른 것들은 모두 필터링 해줍니다.

```python
BLOCK_EXT = re.compile(r".*\.(php|jsp|asp|exe|sh|cgi|html)$", re.IGNORECASE)
BLOCK_TRAVERSAL = re.compile(r"\.\.|\/etc\/|\/usr\/|\/var\/|\\\\|%2e%2e", re.IGNORECASE)

# 의심스러운 파라미터명 (실제 사용하는 파라미터 확인 후 조정)
SUSPICIOUS_PARAMS = {"file", "path", "include", "template", "doc"}
```

간단하게 middleware에서 검증 후 막거나 통과하게 처리 했습니다.

```python
# Path 검사 (항상)
if BLOCK_EXT.search(path):
    logger.warning(f"BLOCKED (path) - {real_ip} - {request.method} {path}")
    return Response(status_code=410)

# 디렉토리 트래버설 검사 (전체 URL)
full_url = str(request.url)
if BLOCK_TRAVERSAL.search(full_url):
    logger.warning(f"BLOCKED (traversal) - {real_ip} - {full_url}")
    return Response(status_code=410)
```

### Middleware 전체 코드

로그 수집을 위해 Kafka로 넘기고 있습니다.

```python
import json
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo
from starlette.middleware.base import BaseHTTPMiddleware
from util.kafka_producer import produce_message
import logging
import time
from dotenv import load_dotenv
import os
import re
from fastapi.responses import Response
from collections import defaultdict

BLOCK_EXT = re.compile(r".*\.(php|jsp|asp|exe|sh|cgi|html)$", re.IGNORECASE)
BLOCK_TRAVERSAL = re.compile(r"\.\.|\/etc\/|\/usr\/|\/var\/|\\\\|%2e%2e", re.IGNORECASE)

# 의심스러운 파라미터명 (실제 사용하는 파라미터 확인 후 조정)
SUSPICIOUS_PARAMS = {"file", "path", "include", "template", "doc"}

# 화이트리스트 경로 (필요시 추가)
SAFE_PATHS = {"/download", "/api/files"}

logger = logging.getLogger("api.access")
load_dotenv()


class KafkaProducerMiddleware(BaseHTTPMiddleware):
    request_counts = defaultdict(list)

    async def dispatch(self, request, call_next):
        start_time = time.time()
        path = request.url.path

        real_ip = (
            request.headers.get("cf-connecting-ip")
            or request.headers.get("x-real-ip")
            or request.headers.get("x-forwarded-for", "").split(",")[0].strip()
            or request.client.host
            if request.client
            else "unknown"
        )

        # 1. Path 검사 (항상)
        if BLOCK_EXT.search(path):
            logger.warning(f"BLOCKED (path) - {real_ip} - {request.method} {path}")
            return Response(status_code=410)

        # 2. 화이트리스트가 아닌 경우만 쿼리 검사
        if path not in SAFE_PATHS:
            # 의심스러운 파라미터만 검사
            for param_name in SUSPICIOUS_PARAMS:
                param_value = request.query_params.get(param_name, "")
                if BLOCK_EXT.search(param_value) or BLOCK_TRAVERSAL.search(param_value):
                    logger.warning(
                        f"BLOCKED (param) - {real_ip} - {request.method} {path}?"
                        f"{param_name}={param_value}"
                    )
                    return Response(status_code=410)

        # 3. 디렉토리 트래버설 검사 (전체 URL)
        full_url = str(request.url)
        if BLOCK_TRAVERSAL.search(full_url):
            logger.warning(f"BLOCKED (traversal) - {real_ip} - {full_url}")
            return Response(status_code=410)

        # Rate limiting
        now = datetime.now()
        self.request_counts[real_ip] = [
            t for t in self.request_counts[real_ip] if now - t < timedelta(minutes=1)
        ]

        if len(self.request_counts[real_ip]) > 100:
            logger.warning(f"Rate limit exceeded: {real_ip}")
            return Response(status_code=429)

        self.request_counts[real_ip].append(now)

        response = await call_next(request)
        process_time = time.time() - start_time

        if real_ip != os.getenv("IP"):
            logger.info(
                f'{real_ip} - "{request.method} {path}" '
                f"{response.status_code} - {process_time:.3f}s"
            )
            data = {
                "method": request.method,
                "link": path,
                "footprint_time": datetime.now(ZoneInfo("Asia/Seoul")).isoformat(),
            }
            produce_message(json.dumps(data))

        return response

```
