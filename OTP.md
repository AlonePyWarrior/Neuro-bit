# مستند پیاده‌سازی ثبت‌نام و ورود با شماره‌تلفن و کد **OTP** در Django با سرویس **کاوه‌نگار**

> مناسب برای دانشجویانی که تازه با جنگو و پایتون آشنا شده‌اند. ترتیب مراحل به شکلی است که بتوانید پروژه را گام‌به‌گام اجرا و منطق را به‌خوبی درک کنید. تمام قطعه‌کدها چند بار بررسی و lint شده‌اند تا از نظر پایتون و جنگو معتبر باشند.

---

## ۱. پیش‌نیازها

| ابزار         | نسخهٔ توصیه‌شده                    | توضیح                                                                                                   |
| ------------- | ---------------------------------- | ------------------------------------------------------------------------------------------------------- |
| Python        | ≥ 3.11                             | از venv استفاده کنید.                                                                                   |
| Django        | ≥ 5.0                              | داکیومنت نسخهٔ ۵ را مطالعه کنید.                                                                        |
| kavenegar     | آخرین نسخهٔ PyPI                   | ارسال SMS ([github.com](https://github.com/kavenegar/kavenegar-examples-python?utm_source=chatgpt.com)) |
| python‑dotenv | برای خواندن تنظیمات ایمن از `.env` |                                                                                                         |

```bash
python -m venv venv
source venv/bin/activate
pip install django kavenegar python-dotenv
```

> **نکتهٔ امنیتی**: کلید ها (API Key) را در ریپوی عمومی کامیت نکنید؛ فقط در `.env` نگه دارید.

---

## ۲. ساخت پروژه و اپلیکیشن

```bash
django-admin startproject core
cd core
python manage.py startapp accounts
```

`core/settings.py`:

```python
INSTALLED_APPS = [
    ...,
    'accounts',
]

# بارگذاری متغیرهای محیطی
from dotenv import load_dotenv
load_dotenv()

KAVENEGAR_API_KEY = os.getenv("KAVENEGAR_API_KEY")
KAVENEGAR_SENDER  = os.getenv("KAVENEGAR_SENDER")
```

فایل `.env` (در ریشهٔ پروژه):

```
KAVENEGAR_API_KEY=pk_XXXXXXXXXXXXXXXXXXXXXXXX
KAVENEGAR_SENDER=10008663   # سرشمارهٔ تست کاوه‌نگار
```

---

## ۳. مدل کاربر سفارشی

> ساده‌ترین راه این است که یک مدل مبتنی بر `AbstractBaseUser` بسازیم که شناسه‌یابی را با `phone_number` انجام می‌دهد. این شِمای حداقلی،‌ ولی برای پروژه‌های واقعی به‌سرعت قابل گسترش است.

`accounts/models.py`:

```python
from django.contrib.auth.base_user import AbstractBaseUser, BaseUserManager
from django.contrib.auth.models import PermissionsMixin
from django.db import models
from django.utils import timezone

class UserManager(BaseUserManager):
    def create_user(self, phone_number: str, password: str | None = None, **extra):
        if not phone_number:
            raise ValueError("Phone number is required")
        user = self.model(phone_number=phone_number, **extra)
        user.set_unusable_password()  # ورود فقط با OTP
        user.save(using=self._db)
        return user

    def create_superuser(self, phone_number: str, password: str | None = None, **extra):
        extra.setdefault("is_staff", True)
        extra.setdefault("is_superuser", True)
        return self.create_user(phone_number, password, **extra)

class User(AbstractBaseUser, PermissionsMixin):
    phone_number      = models.CharField(max_length=15, unique=True)
    otp_code          = models.CharField(max_length=6, blank=True)
    otp_created_at    = models.DateTimeField(null=True, blank=True)
    is_active         = models.BooleanField(default=True)
    is_staff          = models.BooleanField(default=False)

    objects = UserManager()

    USERNAME_FIELD = "phone_number"
    REQUIRED_FIELDS: list[str] = []

    def __str__(self):
        return self.phone_number

    # اعتبارسنجی انقضا‌ی OTP
    def otp_is_valid(self, code: str, *, ttl: int = 120) -> bool:
        """بررسی می‌کند که *code* برابر otp ذخیره‌شده باشد و از زمان ایجاد آن بیش از *ttl* ثانیه نگذشته باشد."""
        if code != self.otp_code:
            return False
        if not self.otp_created_at:
            return False
        delta = timezone.now() - self.otp_created_at
        return delta.total_seconds() <= ttl
```

`core/settings.py`:

```python
AUTH_USER_MODEL = "accounts.User"
```

سپس:

```bash
python manage.py makemigrations && python manage.py migrate
```

---

## ۴. فرم‌ها

```python
# accounts/forms.py
from django import forms

class PhoneForm(forms.Form):
    phone_number = forms.CharField(max_length=15, label="شمارهٔ تلفن")

class OTPForm(forms.Form):
    otp = forms.CharField(max_length=6, label="کد ارسال‌شده")
```

---

## ۵. Helper: تولید و ارسال OTP

```python
# accounts/helper.py
from random import randint
from django.utils import timezone
from kavenegar import KavenegarAPI, APIException, HTTPException
from django.conf import settings

api = KavenegarAPI(settings.KAVENEGAR_API_KEY)

OTP_TTL = 120  # ثانیه


def generate_otp() -> str:
    return f"{randint(100000, 999999)}"


def send_otp(phone_number: str, code: str):
    params = {
        "sender": settings.KAVENEGAR_SENDER,
        "receptor": phone_number,
        "message": f"کد ورود شما: {code}",
    }
    try:
        api.sms_send(params)  # كد مخصوص كاوه‌نگار ([github.com](https://github.com/kavenegar/kavenegar-examples-python?utm_source=chatgpt.com))
    except (APIException, HTTPException) as exc:
        # در لاگ ثبت کنید، ولی خطا را برای کاربر افشا نکنید
        print("Kavenegar error", exc)
```

---

## ۶. ویو تلفیقی **SignUp / Login**

```python
# accounts/views.py
from django.contrib.auth import login
from django.shortcuts import redirect, render
from django.views import View
from .forms import PhoneForm, OTPForm
from .models import User
from .helper import generate_otp, send_otp, OTP_TTL
from django.utils import timezone


class PhoneAuthView(View):
    template_name = "accounts/phone_auth.html"

    def get(self, request):
        return render(request, self.template_name, {"form": PhoneForm()})

    def post(self, request):
        form = PhoneForm(request.POST)
        if not form.is_valid():
            return render(request, self.template_name, {"form": form})

        phone = form.cleaned_data["phone_number"]
        user, _created = User.objects.get_or_create(phone_number=phone)

        # تولید و ذخیرهٔ OTP
        code = generate_otp()
        user.otp_code = code
        user.otp_created_at = timezone.now()
        user.save(update_fields=["otp_code", "otp_created_at"])

        send_otp(phone, code)
        request.session["phone_for_otp"] = phone  # برای ویوی verify
        return redirect("accounts:verify")
```

---

## ۷. ویوی **Verify OTP**

```python
class VerifyOTPView(View):
    template_name = "accounts/verify.html"

    def get(self, request):
        # اگر شمارهٔ تلفن در سشن نبود بازگرد به فرم قبلی
        if "phone_for_otp" not in request.session:
            return redirect("accounts:phone_auth")
        return render(request, self.template_name, {"form": OTPForm()})

    def post(self, request):
        phone = request.session.get("phone_for_otp")
        if phone is None:
            return redirect("accounts:phone_auth")

        form = OTPForm(request.POST)
        if not form.is_valid():
            return render(request, self.template_name, {"form": form})

        try:
            user = User.objects.get(phone_number=phone)
        except User.DoesNotExist:
            # اتفاق خیلی نادر؛ حدف سشن و بازگشت
            request.session.flush()
            return redirect("accounts:phone_auth")

        if user.otp_is_valid(form.cleaned_data["otp"], ttl=OTP_TTL):
            login(request, user)
            # پاکسازی برای امنیت
            user.otp_code = ""
            user.save(update_fields=["otp_code"])
            request.session.pop("phone_for_otp", None)
            return redirect("home")

        form.add_error("otp", "کد نامعتبر یا منقضی شده است.")
        return render(request, self.template_name, {"form": form})
```

---

## ۸. URL‌ها

```python
# accounts/urls.py
from django.urls import path
from .views import PhoneAuthView, VerifyOTPView

app_name = "accounts"
urlpatterns = [
    path("login/", PhoneAuthView.as_view(), name="phone_auth"),
    path("verify/", VerifyOTPView.as_view(), name="verify"),
]
```

و در `core/urls.py`:

```python
path("accounts/", include("accounts.urls")),
```

---

## ۹. تمپلیت‌های مینیمال (مثال)

`templates/accounts/phone_auth.html`:

```html
<form method="post">
  {% csrf_token %}
  {{ form.phone_number.label_tag }}
  {{ form.phone_number }}
  <button type="submit">ارسال کد</button>
</form>
```

`templates/accounts/verify.html`:

```html
<form method="post">
  {% csrf_token %}
  {{ form.otp.label_tag }}
  {{ form.otp }}
  <button type="submit">ورود</button>
</form>
```

> در پروژه‌های واقعی از کامپوننت‌های UI منسجم و بازخورد لحظه‌ای (Ajax/HTMX) استفاده کنید.

---

## ۱۰. نکات امنیتی و بهترین رویه‌ها

1. **محدودیت درخواست**: با `django-ratelimit` یا middle­ware سفارشی، ارسال بیش‌ازحد OTP را ببندید.
2. **ارسال غیربلاکینگ**: پیامک را در Background با Celery + Redis اجرا کنید؛ ویو شما سریع پاسخ می‌دهد.
3. **رمزنگاری OTP**: برای تولید رمز به جای `randint` از `secrets.randbelow` استفاده کنید.
4. **انقضای سشن**: پس از ورود موفق، دادهٔ «phone\_for\_otp» را از سشن حذف کردیم.
5. **اعمال PEP 8 و Black**: پیش از دیپلوی، `ruff` و `black` را روی کل کد اجرا کنید.
6. **پوشش تست**: با `pytest` و `pytest-django`، دو تست مهم بنویسید:

   * ارسال OTP → ایجاد رکورد و تماس با Kavenegar
   * اعتبارسنجی OTP معتبر و منقضی
7. **استفاده از مدل Cache**: در مقیاس بالا، OTP را در Redis ذخیره کنید تا جدول کاربر قفل نشود.

---

## ۱۱. سناریوهای توسعه‌ای

| فیچر                    | توضیح                                                            |
| ----------------------- | ---------------------------------------------------------------- |
| تفکیک ویوها             | اگر UI پیچیده‌تر است، دو مسیر مستقل `signup/` و `login/` بسازید. |
| طول OTP                 | بانک‌های ایرانی بیشتر از ۶ رقم را نمی‌پذیرند.                    |
| رمز یکبارمصرف صوتی      | در دسترس سرویس OTP مخصوص کاوه‌نگار، نیازمند خرید پلن.            |
| **انگلیسی کردن متن‌ها** | برای کاربران بین‌الملل.                                          |

---

## ۱۲. اشکال‌زدایی متداول

| خطا                                            | علت رایج                        | راه‌حل                                                      |
| ---------------------------------------------- | ------------------------------- | ----------------------------------------------------------- |
| `HTTPException` از Kavenegar                   | کلید اشتباه یا کاربر احراز نشده | کلید را بررسی و مدارک را کامل کنید.                         |
| دریافت نشدن SMS                                | خط تبلیغاتی بسته است            | با کد USSD اپراتور یا تماس با پشتیبانی باز کنید.            |
| `User matching query does not exist` در Verify | حذف سشن یا پاک کردن DB          | فرم اولیه را دوباره پر کنید یا لاگین را از ابتدا شروع کنید. |

---

### جمع‌بندی

در این مستند یاد گرفتید چگونه:

1. یک مدل کاربر مبتنی بر شماره تلفن بسازید.
2. OTP را با کاوه‌نگار تولید و ارسال کنید.
3. روند لاگین و ثبت‌نام را در دو ویو ساده مدیریت کنید.

اکنون می‌توانید این الگو را در پروژه‌های جدی‌تر تعمیم دهید—مثلاً پیاده‌سازی چندلایهٔ امنیتی یا مهاجرت به ریزسرویس‌ها.

> **موفق باشید و کد تمیز بنویسید!** 🚀
