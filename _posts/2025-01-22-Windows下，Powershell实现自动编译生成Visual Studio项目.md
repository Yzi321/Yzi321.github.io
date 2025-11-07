---
layout:     post
title:      Windows下，Powershell实现自动编译生成Visual Studio项目
subtitle:   Powershell脚本自动编译Visual Studio项目；脚本自动更新版本号
date:       2025-01-22
author:     Yzi321
header-img: img/post-bg-keybord.jpg
catalog: true
tags:
    - Powershell
    - Visual Studio
---


> windows平台、VS2019、x64、C++/C

在进行生成编译版本，并输出版本时，常常会遇到多个工程编译，同时提取出所需的动态库dll、执行文件exe，并进行打包。
每次进行编译和打包均需要手动操作，过于繁琐，所以这里通过一个`powershell`文件去执行自动编译，后续可以通过`批处理`或者其他的`powershell`文件实现打包和发布。
同时这里也分享一个`自动版本变更`的脚本。

### 一、Visual Studio 命令行编译项目

##### 1、MakeRelease.ps1文件
注意：`$outDir`、`$tempDir`的路径是`.vcxproj`项目文件所在的路径为基准的；`$projectPath`的路径是以当前ps1文件的路径为基准的；
``` powershell
# 设置 MSBuild 路径
$msbuildPath = "D:\VS2019\MSBuild\Current\Bin\MSBuild.exe"

# 设置项目文件路径
$projectPath = ".\..\YourProjectName.vcxproj"

# 设置编译参数
$configuration = "Release"
$platform = "x64"
$outDir = ".\..\x64\Release\"        # 设置编译输出目录
$tempDir = "x64\Release\"   # 设置中间文件存储目录

# 检查 MSBuild 是否存在
if (-Not (Test-Path $msbuildPath)) {
    Write-Error "MSBuild.exe not found at $msbuildPath"
    exit 1
}

# 检查项目文件是否存在
if (-Not (Test-Path $projectPath)) {
    Write-Error "Project file not found at $projectPath"
    exit 1
}

# 构造命令参数
$arguments = @(
    $projectPath,
    "/p:Configuration=$configuration",
    "/p:Platform=$platform",
    "/p:OutDir=$outDir",
    "/p:IntDir=$tempDir",
    "/v:q",
    "/m"
)

# 执行 MSBuild
Write-Output "Starting build..."
Invoke-Expression "$msbuildPath $arguments"

# 检查退出码
if ($LASTEXITCODE -eq 0) {
    Write-Output "Build succeeded. Check output in $outDir"
} else {
    Write-Error "Build failed with exit code $LASTEXITCODE. Detailed output: $output"
}

Write-Output "Build completed."

```

##### 2、执行
``` powershell
./MakeRelease.ps1
```

### 二、版本变更
##### 代码文件：
- (1）VersionUpdate.bat
- (2）VersionUpdate.ps1
- (3）VersionUpdateToSulution.ps1
##### 1、VersionUpdate.bat
```bash
@echo off
chcp 65001

rem 获取当前脚本所在目录
set SCRIPT_DIR=%~dp0

rem 指定 PowerShell 脚本文件名
set PS_SCRIPT=VersionUpdate.ps1

rem 检查 PowerShell 脚本是否存在
if not exist "%SCRIPT_DIR%%PS_SCRIPT%" (
    echo PowerShell script not found at "%SCRIPT_DIR%%PS_SCRIPT%".
    pause
    exit /b 1
)

rem 运行 PowerShell 脚本
powershell.exe -NoProfile -ExecutionPolicy Bypass -File "%SCRIPT_DIR%%PS_SCRIPT%" -readOnly False


for /f "tokens=1,* delims==" %%i in ('powershell -ExecutionPolicy Bypass -File "%SCRIPT_DIR%%PS_SCRIPT%" -readOnly True') do (
    if "%%i"=="Major" set Major=%%j
    if "%%i"=="Minor" set Minor=%%j
    if "%%i"=="Patch" set Patch=%%j
)

echo Major=%Major%
echo Minor=%Minor%
echo Patch=%Patch%

set Version=%Major%.%Minor%.%Patch%
echo Version: %Version%


echo 确认版本号 [y/n]?
set /p userInput=请输入 y 或 n: 

set EnsureVersion=0
if /i "%userInput%"=="y" (
    set EnsureVersion=1
    echo 版本号已确认: %Version%
) else if /i "%userInput%"=="n" (
    echo 版本号未确认，操作已取消.
    exit /b
) else (
    echo 无效输入，操作已取消.
    exit /b
)

REM 更新版本到解决方案

set PS_SCRIPT=VersionUpdateToSolution.ps1

rem 运行 PowerShell 脚本
powershell.exe -NoProfile -ExecutionPolicy Bypass -File "%SCRIPT_DIR%%PS_SCRIPT%" -Major %Major% -Minor %Minor% -Patch %Patch% -Version %Version%
```

##### 2、VersionUpdate.ps1
```powershell
param (
    [string]$readOnly  # 获取传递的输出文件夹路径
)

# 设置版本号文件的路径
$VersionFile = "$PSScriptRoot\version.txt"

# 初始化版本号
function Initialize-Version {
    if (-Not (Test-Path $VersionFile)) {
        Write-Output "Version file not found. Creating with initial version 1.0.0..."
        Set-Content -Path $VersionFile -Value "1.0.0"
    }
}

# 读取版本号
function Get-Version {
    if (Test-Path $VersionFile) {
        return Get-Content -Path $VersionFile -ErrorAction Stop
    } else {
        Write-Error "Version file not found."
        return $null
    }
}

# 更新版本号
function Update-Version {
    param (
        [ValidateSet("Major", "Minor", "Patch")]
        [string]$IncrementType
    )

    $currentVersion = Get-Version
    if (-Not $currentVersion) {
        return
    }

    $parts = $currentVersion -split '\.'
    $Major = [int]$parts[0]
    $Minor = [int]$parts[1]
    $Patch = [int]$parts[2]

    switch ($IncrementType) {
        "Major" {
            $Major++
            $Minor = 0
            $Patch = 0
        }
        "Minor" {
            $Minor++
            $Patch = 0
        }
        "Patch" {
            $Patch++
        }
    }

    $newVersion = "$Major.$Minor.$Patch"
    Set-Content -Path $VersionFile -Value $newVersion
    Write-Output "Version updated to: $newVersion"
}

# 传递版本号结果
function Export-Version {
    $currentVersion = Get-Version
    if (-Not $currentVersion) {
        return
    }

    $parts = $currentVersion -split '\.'
    $Major = $parts[0]
    $Minor = $parts[1]
    $Patch = $parts[2]

    # 输出为 PowerShell 环境变量
    Write-Output "Major=$Major"
    Write-Output "Minor=$Minor"
    Write-Output "Patch=$Patch"
}

# 主逻辑
if ($readOnly -eq "True") {
    # 导出版本号
    Export-Version

} else{
    Write-Output "Initializing version system..."
    Initialize-Version

    Write-Output "Current version: $(Get-Version)"

    # 选择更新版本号的类型
    $choice = Read-Host "Enter version increment type (Major/Minor/Patch, or skip)"
    if ($choice -ne "skip") {
        Update-Version -IncrementType $choice
    }
}
```

##### 3、VersionUpdateToSulution.ps1
```powershell
param (
    [int]$Major,
    [int]$Minor,
    [int]$Patch,
    [string]$Version
)

# 打印参数值
Write-Output "Modify the relevant version information in the project file."
Write-Output "Major: $Major, Minor: $Minor, Patch: $Patch"
Write-Output "Version: $Version"

# 修改项目内部版本号

# 文件路径
$filePath = "./../MyProject/version_time.h"
# 逐行读取文件内容
$fileContent = Get-Content $filePath

# 处理每一行，只修改包含版本号的行
$fileContent = $fileContent | ForEach-Object {
    # 如果当前行包含 'v' ，则进行替换
    if ($_ -match 'v\d+\.\d+\.\d+') {
        # 使用新版本号替换旧版本号，保持 'v' 不变
        $_ -replace 'v\d+\.\d+\.\d+', "v${Version}"
    }
    else {
        # 如果当前行不包含版本号，则保持不变
        $_
    }
}

# 将修改后的内容写回文件
Set-Content $filePath $fileContent
Write-Output "Version updated successfully in $filePath"
```

##### 4、运行
```
./VersionUpdate.bat
```
或直接双击文件即可运行。

- `powershell`环境的变量和`bash`环境内的变量无法直接传递；这里通过`Export-Version`函数传出结果；再使用`param`参数列表传入版本参数
- 版本变更的逻辑是，在脚本目录文件夹下，生成和存储当前版本文件`version.txt`，通过控制它的变更修改版本号，同时再调用`VersionUpdateToSulution.ps1`将版本信息更新到项目代码中