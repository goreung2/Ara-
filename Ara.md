# Ara

**Ara**는 `discord.py` 봇 개발을 위한 데코레이터 중심의 경량 유틸리티
라이브러리입니다.

권한 검사, 서버/DM 제한, 오류 처리, 재시도, 호출 제한, 캐시, 동시 실행
제어, 로그, 실행 시간 측정, 자동 응답, Cog reload 등의 공통 기능을
함수에 간단히 적용할 수 있습니다.

> 이 문서는 현재 Ara 패키지의 실제 구현을 기준으로 작성되었습니다.

------------------------------------------------------------------------

## 목차

-   [설치 및 불러오기](#설치-및-불러오기)
-   [기본 사용법](#기본-사용법)
-   [AraError](#araerror)
-   [오류 처리](#오류-처리)
-   [재시도](#재시도)
-   [로그와 실행 시간](#로그와-실행-시간)
-   [Discord 응답](#discord-응답)
-   [권한 및 실행 위치 제한](#권한-및-실행-위치-제한)
-   [호출 제어](#호출-제어)
-   [캐시](#캐시)
-   [초기화 및 환경 설정](#초기화-및-환경-설정)
-   [Cog 관리](#cog-관리)
-   [설정 파일](#설정-파일)
-   [기능 요약](#기능-요약)
-   [주의사항](#주의사항)

------------------------------------------------------------------------

## 설치 및 불러오기

Ara의 공개 API는 다음과 같습니다.

``` python
from Ara import Ara, AraError
```

`Ara` 클래스에는 모든 데코레이터가 정적 메서드로 등록되어 있습니다.

``` python
@Ara.retry(retries=3)
async def request():
    ...
```

개별 모듈에서 직접 가져오는 것도 가능합니다.

``` python
from Ara.retry import retry
```

Discord 관련 기능을 사용하려면 `discord.py`가 설치되어 있어야 합니다.
`is_owner`, `is_admin`, `is_guild`는 `discord.ext.commands`가 존재할
경우 Discord의 `commands.check` 방식으로 동작합니다.

------------------------------------------------------------------------

# 기본 사용법

Ara의 핵심은 데코레이터입니다.

``` python
@Ara.cooldown(5)
async def ping(ctx):
    await ctx.send("Pong!")
```

여러 기능도 조합할 수 있습니다.

``` python
@Ara.catch()
@Ara.cooldown(5)
@Ara.has_permissions(manage_messages=True)
async def clear(ctx):
    ...
```

데코레이터를 여러 개 사용할 때는 **아래쪽에 가까운 데코레이터부터 실제
함수에 적용**된다는 Python의 데코레이터 규칙을 고려해야 합니다.

------------------------------------------------------------------------

# AraError

`AraError`는 Ara가 사용자에게 전달하기 적합한 오류를 표현할 때 사용하는
기본 예외입니다.

``` python
raise AraError("권한이 없습니다.")
```

원본 예외를 함께 저장할 수도 있습니다.

``` python
try:
    ...
except Exception as error:
    raise AraError(
        "작업에 실패했습니다.",
        original=error,
    )
```

속성:

  속성         설명
  ------------ --------------------------------
  `reason`     오류 메시지
  `original`   원래 발생한 예외. 없을 수 있음

------------------------------------------------------------------------

# 오류 처리

## `@Ara.catch`

함수 실행 중 지정한 예외를 잡아 Discord 사용자에게 자동으로 안내합니다.

``` python
@Ara.catch()
async def buy(ctx, item):
    ...
```

특정 예외만 처리할 수도 있습니다.

``` python
@Ara.catch(
    ValueError,
    message="입력값이 올바르지 않습니다.",
)
async def divide(ctx, value):
    ...
```

### 옵션

``` python
@Ara.catch(
    message="오류가 발생했습니다.",
    reraise=False,
    ephemeral=False,
    log=True,
)
async def command(ctx):
    ...
```

  옵션            기본값        설명
  --------------- ------------- -----------------------------------------
  `*exceptions`   `Exception`   처리할 예외 종류
  `message`       `None`        일반 예외에 사용할 메시지
  `reraise`       `False`       안내 후 예외를 다시 발생시킬지 여부
  `ephemeral`     `False`       Interaction 응답을 비공개로 보낼지 여부
  `log`           `True`        예외를 logger에 기록할지 여부

`AraError`가 발생하면 `message`보다 `AraError.reason`이 우선 사용됩니다.

`Context`와 `Interaction` 모두 지원합니다.

> `@Ara.catch`는 async 함수에서만 사용할 수 있습니다.

------------------------------------------------------------------------

## `@Ara.ignore_error`

지정한 예외를 무시하고 `default` 값을 반환합니다.

``` python
@Ara.ignore_error(ValueError, default=False)
async def check():
    ...
```

예외를 지정하지 않으면 일반 `Exception`을 처리합니다.

``` python
@Ara.ignore_error
async def task():
    ...
```

``` python
@Ara.ignore_error(
    ValueError,
    TypeError,
    default=None,
)
def parse():
    ...
```

  옵션            설명
  --------------- ----------------------------------
  `*exceptions`   무시할 예외 종류
  `default`       예외 발생 시 반환값. 기본 `None`

비동기 함수에서는 `asyncio.CancelledError`를 무시하지 않고 다시
발생시킵니다.

Sync/Async 함수 모두 지원합니다.

------------------------------------------------------------------------

# 재시도

## `@Ara.retry`

함수 실행이 지정한 예외로 실패하면 다시 실행합니다.

``` python
@Ara.retry(retries=3)
async def request_api():
    ...
```

### 옵션

``` python
@Ara.retry(
    retries=5,
    delay=1,
    backoff=True,
    exceptions=(ConnectionError,),
)
async def connect():
    ...
```

  옵션           설명
  -------------- -----------------------------------------
  `retries`      최대 실행 횟수
  `delay`        재시도 전 대기 시간(초)
  `backoff`      지수 백오프 사용 여부
  `exceptions`   재시도할 예외 종류. 기본 `(Exception,)`

`backoff=True`이면 대기 시간이 `delay × 2^(시도번호-1)` 형태로
증가합니다.

예를 들어 `delay=1`이면 재시도 전 대기는 다음과 같이 증가합니다.

``` text
1초 → 2초 → 4초 → 8초 ...
```

최종 시도까지 실패하면 `AraError`가 발생하며 원본 예외가 `original`에
저장됩니다.

`asyncio.CancelledError`는 재시도하지 않습니다.

Sync/Async 함수 모두 지원합니다.

------------------------------------------------------------------------

# 로그와 실행 시간

## `@Ara.log`

함수의 실행, 성공, 실패를 Python `logging`으로 기록합니다.

``` python
@Ara.log()
async def task():
    ...
```

기본 로그 레벨은 `logging.INFO`입니다.

``` python
@Ara.log(logging.DEBUG)
async def task():
    ...
```

로그 이름도 지정할 수 있습니다.

``` python
@Ara.log(
    logging.INFO,
    name="my_bot",
)
async def task():
    ...
```

실패 시 traceback을 포함한 예외 로그가 기록되고 예외는 다시 발생합니다.

Sync/Async 함수 모두 지원합니다.

------------------------------------------------------------------------

## `@Ara.measure`

함수 실행 시간을 측정합니다.

``` python
@Ara.measure
async def expensive_task():
    ...
```

또는 로그를 끌 수 있습니다.

``` python
@Ara.measure(log=False)
async def task():
    ...
```

로그에는 약 다음과 같은 정보가 기록됩니다.

``` text
[Ara] expensive_task 실행시간: 0.1234s
```

`time.perf_counter()`를 사용해 실행 시간을 측정하며, 함수가 성공하든
예외가 발생하든 측정 로그가 남습니다.

Sync/Async 함수 모두 지원합니다.

------------------------------------------------------------------------

# Discord 응답

## `@Ara.typing`

Discord의 typing 상태를 함수 실행 동안 표시합니다.

``` python
@Ara.typing
async def generate(ctx):
    ...
```

`typing()`과 `send()`를 가진 객체를 함수 인자에서 찾아 사용합니다.

대상을 찾지 못하면 typing 표시 없이 함수를 실행합니다.

> async 함수에서만 사용할 수 있습니다.

------------------------------------------------------------------------

## `@Ara.auto_reply`

함수가 반환한 값을 자동으로 Discord에 전송합니다.

``` python
@Ara.auto_reply()
async def hello(ctx):
    return "안녕하세요!"
```

반환값이 `None`이면 아무것도 전송하지 않습니다.

### Interaction

Interaction에서는 아직 최초 응답을 하지 않았다면
`response.send_message()`를 사용하고, 이미 응답했다면
`followup.send()`를 사용합니다.

``` python
@Ara.auto_reply(ephemeral=True)
async def secret(interaction):
    return "비공개 메시지"
```

### Context

`send()`와 `message` 속성을 가진 Context를 찾아 `ctx.send()`를
호출합니다.

Discord Context를 찾지 못하면 `AraError`가 발생합니다.

> async 함수에서만 사용할 수 있습니다.

------------------------------------------------------------------------

## `@Ara.confirm`

함수를 실행하기 전에 사용자에게 확인을 요청합니다.

``` python
@Ara.confirm("정말 삭제하시겠습니까? (yes/no)")
async def delete(ctx):
    ...
```

확인으로 인정되는 입력:

``` text
yes
y
네
예
```

취소로 인정되는 입력:

``` text
no
n
아니오
아니요
```

### 옵션

``` python
@Ara.confirm(
    prompt="정말 진행하시겠습니까?",
    timeout=30,
    cancel_message="취소되었습니다.",
    timeout_message="시간이 초과되었습니다.",
)
async def action(ctx):
    ...
```

  -----------------------------------------------------------------------------------
  옵션                    기본값                              설명
  ----------------------- ----------------------------------- -----------------------
  `prompt`                `정말 진행하시겠습니까? (yes/no)`   확인 요청 문구

  `timeout`               `30.0`                              응답 대기 시간(초)

  `cancel_message`        `취소되었습니다.`                   취소 시 메시지

  `timeout_message`       `시간이 초과되어 취소되었습니다.`   시간 초과 시 메시지
  -----------------------------------------------------------------------------------

응답자는 명령을 실행한 사용자로 제한됩니다.

현재 구현은 `bot.wait_for("message")`를 사용하므로 Context 기반 사용을
전제로 합니다. `ctx.bot` 또는 `client`를 찾지 못하면 확인 절차를
건너뛰고 함수를 실행합니다.

> `timeout`은 0보다 커야 합니다.

------------------------------------------------------------------------

# 권한 및 실행 위치 제한

Ara는 Discord 객체를 함수 인자에서 찾아 권한과 실행 위치를 검사합니다.

------------------------------------------------------------------------

## `@Ara.is_owner`

봇 소유자만 사용할 수 있도록 제한합니다.

``` python
@Ara.is_owner
async def shutdown(ctx):
    ...
```

`discord.py`가 설치된 환경에서는 `ctx.bot.is_owner(ctx.author)`를
사용합니다.

Discord 명령어에서는 `commands.check` 방식으로 동작하며, 실패하면
`commands.CheckFailure`가 발생합니다.

------------------------------------------------------------------------

## `@Ara.is_admin`

Discord의 `Administrator` 권한이 있는 사용자만 실행할 수 있도록
제한합니다.

``` python
@Ara.is_admin
async def admin_command(ctx):
    ...
```

DM에서는 사용할 수 없습니다.

------------------------------------------------------------------------

## `@Ara.is_guild`

서버에서만 사용할 수 있도록 제한합니다.

``` python
@Ara.is_guild
async def server_command(ctx):
    ...
```

DM에서 실행하면 실패합니다.

------------------------------------------------------------------------

## `@Ara.dm_only`

DM에서만 사용할 수 있도록 제한합니다.

``` python
@Ara.dm_only
async def private_command(ctx):
    ...
```

서버에서 실행하면 `AraError`가 발생합니다.

------------------------------------------------------------------------

## `@Ara.nsfw_only`

현재 채널의 `is_nsfw()`가 참일 때만 실행합니다.

``` python
@Ara.nsfw_only
async def restricted_command(ctx):
    ...
```

채널을 찾을 수 없거나 `is_nsfw()`를 사용할 수 없으면 실패합니다.

------------------------------------------------------------------------

## `@Ara.has_role`

사용자가 특정 역할 중 하나를 가지고 있는지 검사합니다.

역할 이름과 ID를 모두 사용할 수 있습니다.

``` python
@Ara.has_role("Moderator")
async def warn(ctx):
    ...
```

``` python
@Ara.has_role(
    123456789012345678,
    "Admin",
)
async def manage(ctx):
    ...
```

여러 역할을 지정하면 **하나라도 일치하면 허용**됩니다.

``` python
@Ara.has_role("Moderator", "Admin")
async def command(ctx):
    ...
```

------------------------------------------------------------------------

## `@Ara.has_permissions`

지정한 Discord 권한을 모두 만족해야 실행할 수 있도록 제한합니다.

``` python
@Ara.has_permissions(
    manage_messages=True,
)
async def clear(ctx):
    ...
```

여러 권한을 동시에 요구할 수 있습니다.

``` python
@Ara.has_permissions(
    kick_members=True,
    ban_members=True,
)
async def moderate(ctx):
    ...
```

`True`로 지정한 권한만 검사합니다. 권한 이름은 `discord.Permissions`의
속성명을 사용합니다.

예:

``` text
manage_messages
kick_members
ban_members
manage_roles
manage_channels
administrator
```

권한이 하나라도 부족하면 `AraError`가 발생하며 누락된 권한 이름이
메시지에 포함됩니다.

------------------------------------------------------------------------

# 호출 제어

## `@Ara.cooldown`

같은 대상의 함수 재실행을 일정 시간 동안 제한합니다.

``` python
@Ara.cooldown(5)
async def ping(ctx):
    await ctx.send("Pong!")
```

### 범위

  `scope`     제한 기준
  ----------- ------------------
  `user`      사용자별. 기본값
  `guild`     서버별
  `channel`   채널별
  `global`    전체

예:

``` python
@Ara.cooldown(30, scope="guild")
async def raid_alert(ctx):
    ...
```

제한 시간이 남아 있으면 `AraError`가 발생합니다.

``` python
@Ara.cooldown(
    10,
    message="10초에 한 번만 사용할 수 있습니다.",
)
async def command(ctx):
    ...
```

> 현재 구현은 async 함수용입니다.

------------------------------------------------------------------------

## `@Ara.debounce`

짧은 시간 안에 연속으로 호출되는 이벤트를 건너뜁니다.

``` python
@Ara.debounce(2)
async def notify(channel):
    ...
```

마지막 허용 호출 이후 지정된 시간이 지나지 않았다면 함수가 실행되지 않고
`None`을 반환합니다.

Sync/Async 함수 모두 지원합니다.

> 현재 구현은 함수별로 하나의 마지막 호출 시간을 관리하므로
> 사용자별/채널별 debounce가 아닙니다.

------------------------------------------------------------------------

## `@Ara.lock`

같은 함수가 동시에 실행되는 것을 막습니다.

``` python
@Ara.lock()
async def buy(ctx, item):
    ...
```

### `scope`

``` python
@Ara.lock(scope="user")
async def buy(ctx):
    ...
```

사용자별 잠금을 사용합니다.

``` python
@Ara.lock(scope="global", wait=True)
async def update_leaderboard():
    ...
```

전체 함수에 하나의 잠금을 사용합니다.

### `wait`

`False`가 기본값입니다.

``` python
@Ara.lock(wait=False)
async def action(ctx):
    ...
```

이미 실행 중이면 즉시 `AraError`가 발생합니다.

`True`로 지정하면 기존 실행이 끝난 후 순서대로 실행합니다.

``` python
@Ara.lock(wait=True)
async def action(ctx):
    ...
```

> 현재 구현은 async 함수용이며 `user`/`global` 두 범위를 지원합니다.

------------------------------------------------------------------------

## `@Ara.rate_limit`

일정 기간 동안 함수가 실행될 수 있는 횟수를 제한합니다.

``` python
@Ara.rate_limit(
    calls=60,
    per=60,
)
async def call_api():
    ...
```

위 예시는 초당이 아니라 **60초 동안 최대 60개의 토큰을 소비할 수
있도록** 제한합니다.

### 옵션

  옵션      설명
  --------- ---------------------------------------------
  `calls`   최대 호출량
  `per`     호출량을 계산하는 기간(초)
  `wait`    한도를 넘었을 때 대기할지 여부. 기본 `True`

한도가 초과되면 `wait=True`에서는 토큰이 다시 생길 때까지 대기합니다.

``` python
@Ara.rate_limit(
    calls=5,
    per=10,
    wait=False,
)
async def send_alert():
    ...
```

`wait=False`에서는 즉시 `AraError`가 발생합니다.

구현은 토큰 버킷 방식이며, Sync/Async 함수 모두 지원합니다.

------------------------------------------------------------------------

# 캐시

## `@Ara.cache`

함수의 반환값을 메모리에 일정 시간 저장합니다.

``` python
@Ara.cache(ttl=300)
async def get_user_profile(user_id):
    return await api.fetch_profile(user_id)
```

같은 인자로 다시 호출했을 때 캐시가 유효하면 원래 함수를 실행하지 않고
저장된 결과를 반환합니다.

### 옵션

  옵션          기본값 설명
  ----------- -------- --------------------------
  `ttl`         `60.0` 캐시 유지 시간(초)
  `maxsize`      `256` 저장 가능한 최대 항목 수

``` python
@Ara.cache(
    ttl=10,
    maxsize=64,
)
def get_config(key):
    return load_config(key)
```

캐시가 가득 차면 가장 오래된 항목을 제거합니다.

캐시는 함수별로 독립된 메모리 저장소를 사용합니다.

캐시를 직접 비울 수도 있습니다.

``` python
get_config.cache_clear()
```

Sync/Async 함수 모두 지원합니다.

### 캐시 키

함수의 positional arguments와 정렬된 keyword arguments를 이용해 키를
만들므로 인자는 hashable해야 합니다.

예를 들어 `int`, `str`, `tuple` 등은 일반적으로 사용할 수 있지만,
`list`, `dict`처럼 hashable하지 않은 값을 인자로 직접 사용하는 경우
문제가 발생할 수 있습니다.

------------------------------------------------------------------------

# 초기화 및 환경 설정

## `@Ara.once`

함수를 최초 한 번만 실제 실행하고 이후 호출에서는 첫 실행 결과를
반환합니다.

``` python
@Ara.once
async def initialize():
    print("초기화")
```

`discord.py`의 `on_ready`처럼 재연결에 의해 여러 번 호출될 수 있는
이벤트의 일회성 초기화에 사용할 수 있습니다.

``` python
@bot.event
@Ara.once
async def on_ready():
    print("최초 준비 완료")
```

Sync/Async 함수 모두 지원합니다.

> 함수 실행 중 예외가 발생하더라도 현재 구현에서는 `called` 상태가 먼저
> 설정되므로 이후 호출이 다시 실행되지 않습니다.

------------------------------------------------------------------------

## `@Ara.require_env`

함수 실행 전에 지정한 환경변수가 모두 존재하고 비어 있지 않은지
검사합니다.

``` python
@Ara.require_env(
    "OPENAI_API_KEY",
    "DATABASE_URL",
)
async def call_ai():
    ...
```

누락된 환경변수가 있으면 `AraError`가 발생합니다.

``` text
다음 환경변수가 설정되지 않았습니다: OPENAI_API_KEY, DATABASE_URL
```

환경변수는 `os.environ.get()`으로 검사합니다.

Sync/Async 함수 모두 지원합니다.

------------------------------------------------------------------------

## `@Ara.deprecated`

기존 함수가 더 이상 권장되지 않음을 `DeprecationWarning`으로 알립니다.

``` python
@Ara.deprecated(
    "v2에서 제거될 예정입니다.",
    replacement="new_command",
)
async def old_command(ctx):
    ...
```

호출 자체는 계속 실행됩니다. 경고만 발생합니다.

  옵션            설명
  --------------- ------------------
  `reason`        사용 중단 사유
  `replacement`   대신 사용할 이름

Sync/Async 함수 모두 지원합니다.

------------------------------------------------------------------------

# Cog 관리

## `@Ara.cog_load`

함수 실행 전에 Discord Bot의 Extension/Cog를 reload합니다.

모든 Extension:

``` python
@Ara.cog_load()
async def reload_all(ctx):
    ...
```

특정 Extension:

``` python
@Ara.cog_load(cogs="admin")
async def reload_admin(ctx):
    ...
```

여러 Extension:

``` python
@Ara.cog_load(
    cogs=["admin", "utility"],
)
async def reload_selected(ctx):
    ...
```

`admin`이라는 이름을 찾았는데 등록된 Extension에 없다면 `cogs.admin`도
확인합니다.

예:

``` text
admin
↓
cogs.admin
```

찾을 수 없는 Cog는 `AraError`를 발생시킵니다.

Reload 과정에서 예외가 발생하면 원본 예외가 `AraError.original`에
저장됩니다.

`cogs="all"`이 아닌 경우 현재 함수의 모듈과 동일한 Extension은
reload하지 않습니다.

> 현재 구현의 wrapper는 async 함수 실행을 전제로 합니다.

------------------------------------------------------------------------

# 설정 파일

Ara는 프로젝트 루트의 `ARA.ini`를 읽어 일부 동작을 설정할 수 있습니다.

파일이 없으면 기본값이 사용됩니다.

## Retry 설정

``` ini
[retry]
default = 3
delay = 1
backoff = true
```

  ------------------------------------------------------------------------
  항목                                        기본값 설명
  --------------------- ---------------------------- ---------------------
  `default`                                      `3` `retry()`에 횟수를
                                                     지정하지 않았을 때의
                                                     최대 실행 횟수

  `delay`                                        `1` 재시도 전 기본 대기
                                                     시간

  `backoff`                                   `true` 기본 지수 백오프 사용
                                                     여부
  ------------------------------------------------------------------------

예를 들어:

``` python
@Ara.retry()
async def request():
    ...
```

은 `ARA.ini`의 retry 설정을 사용합니다.

데코레이터에 직접 값을 전달하면 직접 전달한 값이 우선합니다.

``` python
@Ara.retry(
    retries=5,
    delay=2,
    backoff=False,
)
async def request():
    ...
```

------------------------------------------------------------------------

# 권한 메시지 설정

Ara의 권한 제한 기능은 `ARA.ini`에서 기본 메시지를 변경할 수 있습니다.

``` ini
[permissions]
owner = 봇 소유자만 사용할 수 있습니다.
guild = 서버에서만 사용할 수 있습니다.
admin = 관리자 권한이 필요합니다.
dm = DM에서만 사용할 수 있습니다.
nsfw = NSFW 채널에서만 사용할 수 있습니다.
role = 이 명령어를 사용할 권한이 없습니다.
```

지원되는 기본 키는 다음과 같습니다.

-   `owner`
-   `guild`
-   `admin`
-   `dm`
-   `nsfw`
-   `role`

특정 함수에 대한 메시지도 지정할 수 있습니다.

``` ini
[permissions.my_command]
admin = 이 명령어는 관리자만 사용할 수 있습니다.
```

모듈/정규화된 함수 이름을 이용한 더 구체적인 설정도 사용할 수 있습니다.

``` ini
[permissions.cogs.admin.AdminCog.delete]
admin = 관리자 전용 삭제 명령입니다.
```

Ara는 더 구체적인 설정을 우선적으로 찾습니다.

------------------------------------------------------------------------

# 기능별 지원 여부

  기능                 Sync   Async   Discord 필요
  ------------------- ------ ------- --------------
  `retry`               O       O          X
  `ignore_error`        O       O          X
  `log`                 O       O          X
  `measure`             O       O          X
  `typing`              X       O          O
  `auto_reply`          X       O          O
  `is_owner`            \-      O          O
  `is_guild`            \-      O          O
  `is_admin`            \-      O          O
  `cooldown`            X       O          O
  `dm_only`             X       O          O
  `has_role`            X       O          O
  `has_permissions`     X       O          O
  `nsfw_only`           X       O          O
  `confirm`             X       O          O
  `catch`               X       O          O
  `lock`                X       O          O
  `cache`               O       O          X
  `debounce`            O       O          X
  `once`                O       O          X
  `require_env`         O       O          X
  `deprecated`          O       O          X
  `rate_limit`          O       O          X
  `cog_load`            \-      O          O

`-`는 구현이 Discord의 `commands.check` 또는 async wrapper를 전제로
한다는 의미입니다.

------------------------------------------------------------------------

# 실전 예제

## 관리자 전용 + 쿨다운 + 오류 안내

``` python
from discord.ext import commands
from Ara import Ara


@commands.command()
@Ara.catch()
@Ara.cooldown(5)
@Ara.has_permissions(manage_messages=True)
async def clear(ctx):
    await ctx.channel.purge(limit=10)
    await ctx.send("메시지를 정리했습니다.")
```

------------------------------------------------------------------------

## API 호출 + 재시도 + 캐시

``` python
from Ara import Ara


@Ara.cache(ttl=300)
@Ara.retry(
    retries=3,
    delay=1,
    backoff=True,
)
async def get_profile(user_id: int):
    return await api.fetch_profile(user_id)
```

------------------------------------------------------------------------

## 외부 API 호출량 제한

``` python
@Ara.rate_limit(
    calls=60,
    per=60,
)
async def request_api():
    return await api.request()
```

------------------------------------------------------------------------

## 버튼 연타 및 중복 실행 방지

``` python
@Ara.lock()
async def buy(ctx, item):
    ...
```

------------------------------------------------------------------------

## 서버 전용 명령

``` python
@commands.command()
@Ara.is_guild
async def server_info(ctx):
    await ctx.send(f"서버: {ctx.guild.name}")
```

------------------------------------------------------------------------

## DM 전용 명령

``` python
@commands.command()
@Ara.dm_only
async def private_info(ctx):
    await ctx.send("DM에서만 사용할 수 있는 명령입니다.")
```

------------------------------------------------------------------------

## 역할 기반 명령

``` python
@commands.command()
@Ara.has_role("Moderator", "Admin")
async def announce(ctx):
    await ctx.send("권한이 확인되었습니다.")
```

------------------------------------------------------------------------

## 반환값 자동 응답

``` python
@Ara.auto_reply()
async def hello(ctx):
    return f"안녕하세요, {ctx.author.mention}!"
```

------------------------------------------------------------------------

## 사용자 확인 후 실행

``` python
@commands.command()
@Ara.confirm(
    "정말 이 작업을 실행할까요? (yes/no)",
    timeout=20,
)
async def reset(ctx):
    await ctx.send("작업을 실행합니다.")
```

------------------------------------------------------------------------

# 데코레이터 조합 예시

Ara는 여러 공통 기능을 하나의 명령어에 적용할 수 있습니다.

``` python
@commands.command()
@Ara.catch(ephemeral=True)
@Ara.cooldown(10)
@Ara.lock()
@Ara.has_permissions(manage_messages=True)
@Ara.typing
async def moderate(ctx):
    ...
```

이러한 방식으로 명령어 자체의 핵심 로직과 공통적인 제어 로직을 분리할 수
있습니다.

------------------------------------------------------------------------

# 내부 동작 특성

Ara는 함수의 인자에서 Discord 객체를 자동으로 찾아 기능을 적용합니다.

### Bot 탐색

다음과 같은 객체를 인식합니다.

-   `Context`의 `.bot`
-   `Interaction`의 `.client`
-   `reload_extension`과 `is_owner`를 가진 Bot 객체

### Guild 탐색

`.guild` 속성을 가진 객체를 사용합니다.

### 사용자 탐색

다음 순서로 사용자를 찾습니다.

-   `Context.author`
-   `Interaction.user`
-   `guild_permissions`를 가진 Member 객체

### Channel 탐색

다음과 같은 객체에서 채널을 찾습니다.

-   `.channel`
-   `send`와 `id`를 가진 Channel 객체

따라서 일반적인 `discord.py` Context/Interaction을 함수의 인자로
사용하는 경우 대부분의 Discord 전용 데코레이터를 별도의 설정 없이 사용할
수 있습니다.

------------------------------------------------------------------------

# 주의사항

## 메모리 기반 기능

다음 기능은 현재 Python 프로세스 메모리에 상태를 저장합니다.

-   `cache`
-   `cooldown`
-   `debounce`
-   `lock`
-   `once`
-   `rate_limit`

따라서 봇 프로세스를 재시작하면 해당 상태가 초기화됩니다.

여러 프로세스/워커를 사용하는 환경에서는 각 프로세스가 별도의 상태를
가질 수 있습니다.

------------------------------------------------------------------------

## `cooldown`과 `debounce`의 차이

`cooldown`은 일정 시간 동안 같은 범위의 재실행을 **차단**하고
`AraError`를 발생시킵니다.

``` python
@Ara.cooldown(5)
async def command(ctx):
    ...
```

`debounce`는 짧은 시간 동안의 연속 호출을 **조용히 건너뛰고 `None`을
반환**합니다.

``` python
@Ara.debounce(2)
async def event():
    ...
```

------------------------------------------------------------------------

## `lock`과 `rate_limit`의 차이

`lock`은 여러 실행이 동시에 겹치는 것을 방지합니다.

`rate_limit`은 일정 시간 동안 허용되는 실행량을 제한합니다.

예를 들어:

``` text
lock
→ 동시에 두 번 실행되지 않도록 보호

rate_limit
→ 일정 시간 동안 너무 많이 실행되지 않도록 제한
```

두 기능은 함께 사용할 수도 있습니다.

------------------------------------------------------------------------

## `cache`는 영구 저장소가 아닙니다

`Ara.cache`는 메모리 캐시입니다.

봇 재시작 후 데이터가 유지되어야 하는 경우 데이터베이스나 파일 등의 영구
저장소를 사용해야 합니다.

------------------------------------------------------------------------

## `once`는 프로세스 수명 동안의 1회 실행입니다

`once`는 프로그램이 실행되는 동안 함수의 상태를 메모리에 보관합니다.

프로세스가 종료되거나 다시 시작되면 다시 실행할 수 있습니다.

------------------------------------------------------------------------

## `confirm`은 메시지 응답을 사용합니다

`confirm`은 버튼이나 Discord Component를 사용하는 확인창이 아니라
`message` 이벤트를 기다리는 방식입니다.

확인 입력은 다음 문자열만 인정합니다.

``` text
yes / y / 네 / 예
no / n / 아니오 / 아니요
```

------------------------------------------------------------------------

# 전체 API 목록

`from Ara import Ara`로 사용할 수 있는 공개 API는 다음과 같습니다.

``` text
Ara.AraError
Ara.cog_load
Ara.retry
Ara.ignore_error
Ara.log
Ara.measure
Ara.typing
Ara.auto_reply
Ara.is_owner
Ara.is_guild
Ara.is_admin
Ara.cooldown
Ara.dm_only
Ara.has_role
Ara.has_permissions
Ara.nsfw_only
Ara.confirm
Ara.catch
Ara.lock
Ara.cache
Ara.debounce
Ara.once
Ara.require_env
Ara.deprecated
Ara.rate_limit
```

------------------------------------------------------------------------

# 요약

Ara는 Discord 봇에서 반복적으로 작성되는 공통 로직을 데코레이터 하나로
추가할 수 있도록 설계된 라이브러리입니다.

``` text
┌───────────────────────────────┐
│             Ara               │
├───────────────────────────────┤
│ 오류 처리     catch / retry   │
│ 권한 검사     owner / admin   │
│ 위치 제한     guild / DM      │
│ 역할·권한     role / perms    │
│ 호출 제어     cooldown        │
│              debounce         │
│              rate_limit       │
│              lock             │
│ 성능          cache / measure │
│ 로그          log             │
│ Discord       typing / reply  │
│ 확인          confirm         │
│ 초기화        once            │
│ 환경          require_env     │
│ 유지보수      deprecated      │
│ Cog 관리      cog_load        │
└───────────────────────────────┘
```

각 기능은 독립적으로 사용할 수 있으며 여러 데코레이터를 조합해 하나의
Discord 명령어에 여러 정책을 적용할 수 있습니다.
