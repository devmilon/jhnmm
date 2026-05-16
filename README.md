admin py file

from django.contrib import admin
from .models import *
admin.site.register(SkillModel)
admin.site.register(JobPostModel)
admin.site.register(JobSeekerProfileModel)

froms py file

from django import forms
from .models import *
from django.contrib.auth.forms import UserCreationForm,AuthenticationForm

class CreateUserForm(UserCreationForm):
    class Meta:
        model = CustomuserModel
        fields = ['username','email','display_name','user_type','password1','password2']

class AuthForm(AuthenticationForm):
    class Meta:
        model = CustomuserModel
        fields = ['username','password1']

class RecruiterProfileForm(forms.ModelForm):
    class Meta:
        model = RecruiterProfileModel
        fields = '__all__'
        exclude = ['user']

class JobSeekerProfileForm(forms.ModelForm):
    class Meta:
        model = JobSeekerProfileModel
        fields = '__all__'
        exclude = ['user']


class JobPostForm(forms.ModelForm):
    class Meta:
        model = JobPostModel
        fields = '__all__'
        exclude = ['user']
class SkillForm(forms.ModelForm):
    class Meta:
        model = SkillModel
        fields = '__all__'
        

models py file

from django.db import models
from django.contrib.auth.models import AbstractUser

class CustomuserModel(AbstractUser):

    USER_LIST = [
        ("Job_Seeker","Job_Seeker"),
        ("Recruiter","Recruiter"),
    ]

    display_name = models.CharField(null=True, max_length=50)
    user_type = models.CharField(choices=USER_LIST, max_length=50)


class SkillModel(models.Model):
    name = models.CharField(max_length=50,unique=True)
    
    def __str__(self):
        return self.name or "No Skills"

    
class RecruiterProfileModel(models.Model):
    user = models.OneToOneField(CustomuserModel, on_delete=models.CASCADE)
    name = models.CharField(null=True, max_length=50)
    company = models.CharField(null=True, max_length=50)
    phone = models.CharField(null=True, max_length=50)
    address = models.CharField(null=True, max_length=50)
    image = models.ImageField(upload_to="media/profile", max_length=None,null=True)

    def __str__(self):
        return self.name
    

    
class JobSeekerProfileModel(models.Model):
    user = models.OneToOneField(CustomuserModel, on_delete=models.CASCADE) 
    name = models.CharField(null=True, max_length=50)   
    phone = models.CharField(null=True, max_length=50)
    address = models.CharField(null=True, max_length=50)
    skill_set = models.ManyToManyField(SkillModel,blank=True)
    image = models.ImageField(upload_to="media/profile", max_length=None,null=True)
    resume = models.FileField(upload_to="media/resume", max_length=None,null=True)

    def __str__(self):
        return self.name 
    

class JobPostModel(models.Model):
    CATEGORY=[
        ("Full-Time","Full-Time"),
        ("Part-Time","Part-Time"),
        ("Remote","Remote"),
        ("Hybrid","Hybrid"),
    ]
    user = models.ForeignKey(RecruiterProfileModel, on_delete=models.CASCADE)
    title = models.CharField(null=True, max_length=50)
    opening = models.CharField(null=True, max_length=50)
    description = models.TextField(null=True)
    category = models.CharField(choices=CATEGORY, max_length=50,null=True)
    skill_set = models.ManyToManyField(SkillModel,blank=True)


    def __str__(self):
        return self.title or "Empty"


 
class ApplyModel(models.Model):
    STATUS=[
        ("Pending","Pending"),
        ("ShortListed","ShortListed"),
        ("Rejected","Rejected"),
        ("Hired","Hired"),
    ]
    applicant = models.ForeignKey(JobSeekerProfileModel, on_delete=models.CASCADE)
    job = models.ForeignKey(JobPostModel,on_delete=models.CASCADE)
    status = models.CharField(choices=STATUS, max_length=50,default='Pending')


urls py file 

from django.urls import path
from .views import *
urlpatterns = [
    path('register/',registerPage,name='register'),
    path('',loginPage,name='login'),
    path('dashboard/',dashboardPage,name='dashboard'),
    path('recruiter/',recruiterProfilePage,name='recruiter'),
    path('jobSeeker/',jobSeekerProfilePage,name='jobSeeker'),
    path('jobPost/',jobPostPage,name='jobPost'),
    path('addSkill/',skillPage,name='addSkill'),
    path('logout/',logoutPage,name='logout'),
    path('applyJob/<int:id>/',applyJobPage,name='applyJob'),
    path('shortListPage/<int:id>/',shortListPage,name='shortListPage'), 
    path('rejectPage/<int:id>/',rejectPage,name='rejectPage'), 
    path('applicantList/',applicantList,name='applicantList'),
    path('myApplicantions/',myApplicantions,name='myApplicantions'),
    path('skillMatchingPage/',skillMatchingPage,name='skillMatchingPage'),

]


main urls py file


from django.contrib import admin
from django.urls import path,include
from django.contrib import admin
from django.conf import settings
from django.conf.urls.static import static
from django.views.static import serve
from django.urls import path, include, re_path

urlpatterns = [
    path("admin/", admin.site.urls),
    path('',include('job_portal_againApp.urls'))
]
urlpatterns+=re_path(r'^static/(?P<path>.*)$', serve, {'document_root': settings.STATIC_ROOT}),
urlpatterns+=re_path(r'^media/(?P<path>.*)$', serve, {'document_root': settings.MEDIA_ROOT}),


views .py filer

from django.shortcuts import redirect,render
from .models import *
from .forms import *
from django.contrib import messages
from django.contrib.auth import login,logout
from django.contrib.auth.decorators import login_required


def registerPage(request):

    if request.method == "POST":
        form = CreateUserForm(request.POST)
        if form.is_valid():
            form.save()
            messages.success(request,"Account Created Successfully")
            return redirect('login')

    form = CreateUserForm()
    context = {
        'form':form,
        'btn':"Register",
        'form_title':"Create New Account",
    }
    return render(request,"auth/baseForm.html",context)

def loginPage(request):

    if request.method == "POST":
        form = AuthForm(request,request.POST)
        if form.is_valid():
            user = form.get_user()
            login(request,user)
            messages.success(request,"Logged in Successfully")
            return redirect("dashboard")

    form = AuthForm()
    context = {
        'form':form,
        'btn':"Login",
        'form_title':"Login Here",
    }
    return render(request,"auth/baseForm.html",context)
def logoutPage(request):
    logout(request)
    return redirect("login")
@login_required
def dashboardPage(request):

    job_data = JobPostModel.objects.all()


    context = {
        'job_data':job_data
    }

    return render(request,'pages/dashboard.html',context)

@login_required
def recruiterProfilePage(request):
    try:
        recruiter = RecruiterProfileModel.objects.get(user=request.user)
    except RecruiterProfileModel.DoesNotExist:
        recruiter = RecruiterProfileModel.objects.create(user=request.user)

    if request.method == "POST":
        form = RecruiterProfileForm(request.POST,request.FILES, instance=recruiter)
        if form.is_valid():
            form.save()
            messages.success(request,"Profile Updated Successfully")
            return redirect("dashboard")

    form = RecruiterProfileForm(instance=recruiter)
    context = {
        'form':form,
        'btn':"Update",
        'form_title':"Update Recruiter Profile",
    }
    return render(request,'pages/baseForm.html',context)

@login_required
def jobSeekerProfilePage(request):
    try:
        jobSeeker = JobSeekerProfileModel.objects.get(user=request.user)
    except JobSeekerProfileModel.DoesNotExist:
        jobSeeker = JobSeekerProfileModel.objects.create(user=request.user)

    if request.method == "POST":
        form = JobSeekerProfileForm(request.POST,request.FILES, instance=jobSeeker)
        if form.is_valid():
            form.save()
            messages.success(request,"Profile Updated Successfully")
            return redirect("dashboard")

    form = JobSeekerProfileForm(instance=jobSeeker)
    context = {
        'form':form,
        'btn':"Update",
        'form_title':"Update Seeker Profile",
    }
    return render(request,'pages/baseForm.html',context)


@login_required
def jobPostPage(request):
    try:
        recruiter = RecruiterProfileModel.objects.get(user=request.user)
    except RecruiterProfileModel.DoesNotExist:
        recruiter = RecruiterProfileModel.objects.create(user=request.user)

    if request.user.user_type == "Recruiter":
        if request.method =="POST":
            form = JobPostForm(request.POST,request.FILES)
            if form.is_valid():
                data = form.save(commit=False)
                data.user = recruiter
                data.save()
                form.save_m2m() #Important for manyTomanyField
                messages.success(request,'Job Posted')
                return redirect("dashboard")
    form = JobPostForm()
    context={
        'form':form,
        'form_title':"Post A Job",
        'btn':"Post"
    }
    return render(request,"pages/baseForm.html",context)

@login_required
def skillPage(request):
    
   
    if request.method == "POST":
        form = SkillForm(request.POST)
        if form.is_valid():         
            form.save()            
            messages.success(request,'Skill Added')
            return redirect("dashboard")

    form = SkillForm()
    context={
        'form':form,
        'form_title':"Add A Skill",
        'btn':"Add"
    }
    return render(request,"pages/baseForm.html",context)

@login_required
def applyJobPage(request,id):
    job = JobPostModel.objects.get(id=id)
    try:
        applicant = JobSeekerProfileModel.objects.get(user=request.user)
    except JobSeekerProfileModel.DoesNotExist:
        messages.error(request,"Update your profile First")
        return redirect('jobSeeker')

    if request.user.user_type == "Job_Seeker":
        if applicant and job:
            ApplyModel.objects.create(
            applicant = applicant,
            job = job,
            status = 'Pending'
        )
        messages.success(request,"Successfully Applied")
        

    return redirect("dashboard")


@login_required
def applicantList(request):
    try:
        recruiter = RecruiterProfileModel.objects.get(user = request.user)
    except RecruiterProfileModel.DoesNotExist:
        recruiter = RecruiterProfileModel.objects.create(user = request.user)
    if recruiter:
        Apply = ApplyModel.objects.filter(job__user = recruiter)

    context = {
        'Apply':Apply
    }
    return render(request,'pages/list.html',context)

@login_required
def myApplicantions(request):
    try:
        seeker = JobSeekerProfileModel.objects.get(user = request.user)
    except JobSeekerProfileModel.DoesNotExist:
        messages.warning(request,"update Your Profile")
        return redirect('jobSeeker')
    if seeker:
        myApplications = ApplyModel.objects.filter(applicant = seeker)

    context = {
        'myApplications':myApplications
    }
    return render(request,'pages/my_list.html',context)

@login_required
def shortListPage(request,id):

    apply_id = ApplyModel.objects.get(id=id)
    apply_id.status = "ShortListed"
    apply_id.save()

    return redirect('applicantList')

@login_required
def rejectPage(request,id):

    apply_id = ApplyModel.objects.get(id=id)
    apply_id.status = "Rejected"
    apply_id.save()

    return redirect('applicantList')


# skill Match
@login_required
def skillMatchingPage(request):

    try:
        user = JobSeekerProfileModel.objects.get(user = request.user)
    except JobSeekerProfileModel.DoesNotExist:
        return redirect('jobSeeker')   
    if user:
        user_skill =  user.skill_set.all()

        jobs = JobPostModel.objects.filter(skill_set__in = user_skill).distinct()
    
    context={
        'jobs':jobs
    }
    
    return render(request,'pages/Skill_dashbaord.html',context)


setting py file

"""
Django settings for job_portal_again project.

Generated by 'django-admin startproject' using Django 6.0.5.

For more information on this file, see
https://docs.djangoproject.com/en/6.0/topics/settings/

For the full list of settings and their values, see
https://docs.djangoproject.com/en/6.0/ref/settings/
"""

from pathlib import Path

# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent


# Quick-start development settings - unsuitable for production
# See https://docs.djangoproject.com/en/6.0/howto/deployment/checklist/

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = "django-insecure--=i)342b$-a$(+&0bm-t!s+*$nhji+0fe(%c%3n%9_jgyu43&g"

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = False

ALLOWED_HOSTS = ["*"]


# Application definition

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    'whitenoise.runserver_nostatic', 
    "django.contrib.staticfiles",
    'job_portal_againApp',
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    'whitenoise.middleware.WhiteNoiseMiddleware',
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]

ROOT_URLCONF = "job_portal_again.urls"

TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [],
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]

WSGI_APPLICATION = "job_portal_again.wsgi.application"


# Database
# https://docs.djangoproject.com/en/6.0/ref/settings/#databases

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}


# Password validation
# https://docs.djangoproject.com/en/6.0/ref/settings/#auth-password-validators

# AUTH_PASSWORD_VALIDATORS = [
#     {
#         "NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator",
#     },
#     {
#         "NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
#     },
#     {
#         "NAME": "django.contrib.auth.password_validation.CommonPasswordValidator",
#     },
#     {
#         "NAME": "django.contrib.auth.password_validation.NumericPasswordValidator",
#     },
# ]


# Internationalization
# https://docs.djangoproject.com/en/6.0/topics/i18n/

LANGUAGE_CODE = "en-us"

TIME_ZONE = "UTC"

USE_I18N = True

USE_TZ = True


# Static files (CSS, JavaScript, Images)
# https://docs.djangoproject.com/en/6.0/howto/static-files/

STATIC_URL = 'static/'
MEDIA_URL = '/media/'
STATIC_ROOT = BASE_DIR / 'staticfiles/'
MEDIA_ROOT = BASE_DIR / 'media/'
STATICFILES_STORAGE = 'whitenoise.storage.CompressedStaticFilesStorage'
AUTH_USER_MODEL = 'job_portal_againApp.CustomuserModel'
