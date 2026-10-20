brew services start jenkins
http://localhost:8080/

### 终端查看启动的列表
VMRUN="/Applications/VMware Fusion.app/Contents/Public/vmrun"
"$VMRUN" list

### 终端扫描
find /Users -type f -name "*.vmx" 2>/dev/null

- ### 终端执行验证是否成功

### 启动
"/Applications/VMware Fusion.app/Contents/Public/vmrun" -T fusion start "/Users/chen/Virtual Machines.localized/Ubuntu 64-bit Arm 24.04.3.vmwarevm/Ubuntu 64-bit Arm 24.04.3.vmx"  gui


### 列表
"/Applications/VMware Fusion.app/Contents/Public/vmrun" list

### 停止
"/Applications/VMware Fusion.app/Contents/Public/vmrun" -T fusion stop "/Users/chen/Virtual Machines.localized/Ubuntu 64-bit Arm 24.04.3.vmwarevm/Ubuntu 64-bit Arm 24.04.3.vmx" soft

### Jenkins下载插件
Active Choices

### groovy代码

```
def searchDirs = [
    new File("/Users/chen/Virtual Machines.localized"),
    new File("/Users/chen/Documents/Virtual Machines.localized")
]

def vmNames = []

searchDirs.each { dir ->
    if (dir.exists()) {
        dir.eachFileRecurse(groovy.io.FileType.FILES) { file ->
            if (file.name.endsWith(".vmx")) {
                // 去掉 .vmx 后缀，只提取虚拟机名称
                def vmName = file.name.replaceAll(/\.vmx$/, "")
                vmNames.add(vmName)
            }
        }
    }
}

// 排序并去重
return vmNames.sort().unique()
```

<img width="1082" height="930" alt="Image" src="https://github.com/user-attachments/assets/7467162f-e597-4e64-8ffe-a49476f18692" />

<img width="689" height="509" alt="Image" src="https://github.com/user-attachments/assets/a5fb7246-27d0-49d2-9f8d-6567a2b6d896" />

<img width="1082" height="930" alt="Image" src="https://github.com/user-attachments/assets/580a225c-bb86-4004-9664-c0c7da3b3458" />

### shell代码
```
#!/bin/bash
set -e

# VMware Fusion vmrun 命令行工具路径
VMRUN="/Applications/VMware Fusion.app/Contents/Library/vmrun"

if [ ! -f "$VMRUN" ]; then
    echo "[ERROR] 未找到 vmrun 工具: $VMRUN"
    exit 1
fi

echo "=========================================="
echo "1. 解析选中的虚拟机名称"
echo "=========================================="

SELECTED_VMS=()
if [ -n "$VM_LIST" ]; then
    IFS=',' read -r -a RAW_SELECTED <<< "$VM_LIST"
    for name in "${RAW_SELECTED[@]}"; do
        trimmed=$(echo "$name" | xargs)
        if [ -n "$trimmed" ]; then
            SELECTED_VMS+=("$trimmed")
            echo "-> 目标保持运行: $trimmed"
        fi
    done
fi

if [ ${#SELECTED_VMS[@]} -eq 0 ]; then
    echo "提示: 未勾选任何虚拟机，所有运行中的虚拟机将被关机。"
fi

echo "=========================================="
echo "2. 扫描本地所有虚拟机"
echo "=========================================="

ALL_VMS=()
SCAN_DIRS=("/Users/chen/Virtual Machines.localized" "/Users/chen/Documents/Virtual Machines.localized")

for sdir in "${SCAN_DIRS[@]}"; do
    if [ -d "$sdir" ]; then
        while IFS= read -r -d '' file; do
            ALL_VMS+=("$file")
        done < <(find "$sdir" -name "*.vmx" -print0)
    fi
done

echo "共扫描到 ${#ALL_VMS[@]} 台虚拟机。"

echo "=========================================="
echo "3. 获取当前运行状态并同步"
echo "=========================================="

RUNNING_OUTPUT=$("$VMRUN" list)
echo "当前后台运行状态："
echo "$RUNNING_OUTPUT"
echo "------------------------------------------"

# 辅助匹配函数
contains_element() {
    local target="$1"
    shift
    for item in "$@"; do
        if [[ "$item" == "$target" ]]; then
            return 0
        fi
    done
    return 1
}

# 遍历所有扫描到的虚拟机
for vmx in "${ALL_VMS[@]}"; do
    VM_NAME=$(basename "$vmx" .vmx)
    
    # 判断该虚拟机当前是否在运行
    IS_RUNNING=0
    if echo "$RUNNING_OUTPUT" | grep -Fq "$vmx"; then
        IS_RUNNING=1
    fi

    # 判断该虚拟机是否被勾选
    IS_SELECTED=0
    if contains_element "$VM_NAME" "${SELECTED_VMS[@]}"; then
        IS_SELECTED=1
    fi

    if [ $IS_SELECTED -eq 1 ]; then
        if [ $IS_RUNNING -eq 0 ]; then
            echo "[正在启动] $VM_NAME"
            "$VMRUN" -T fusion start "$vmx" gui
        else
            echo "[保持运行] $VM_NAME"
        fi
    else
        if [ $IS_RUNNING -eq 1 ]; then
            echo "[正在关机] $VM_NAME"
            "$VMRUN" -T fusion stop "$vmx" soft || "$VMRUN" -T fusion stop "$vmx" hard
        else
            echo "[保持关闭] $VM_NAME"
        fi
    fi
done

echo "=========================================="
echo "4. 同步完成，最终运行列表："
echo "=========================================="
"$VMRUN" list
```
