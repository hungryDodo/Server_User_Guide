# 中文输入法安装与配置指南（Fcitx5）

## 1. Fcitx5 安装步骤
```bash
# 安装核心组件与输入法引擎
sudo apt install fcitx5 fcitx5-chinese-addons fcitx5-frontend-qt5 fcitx5-frontend-gtk3 fcitx5-frontend-gtk4

# 安装配置工具
sudo apt install fcitx5-config-qt fcitx5-modules kcm-fcitx5

# 重启后执行环境配置
im-config -n fcitx5
```

## 2. Rime 输入法引擎配置
1. 安装 Rime 引擎包：
```bash
sudo apt install fcitx5-rime
```

2. 字典文件配置：
```bash
mkdir -p ~/.local/share/fcitx5/rime
vim ~/.local/share/fcitx5/rime/default.custom.yaml
```
```yaml
# 添加以下内容：
patch:
  schema_list:
    ◦ schema: luna_pinyin        # 明月拼音
  translator/enable_user_dict: true
```

3. 重新部署配置：
```bash
fcitx5-remote -r
```

## 3. 皮肤更换流程
1. 下载主题（以 Material-Color 为例）：
```bash
git clone https://github.com/hosxy/fcitx5-material-color.git \
~/.local/share/fcitx5/themes/material-color
```

2. 修改经典UI配置：
```bash
vim ~/.config/fcitx5/conf/classicui.conf
```
```ini
# 修改以下参数：
Theme=material-color
Vertical Candidate List=False
```

## 4. 候选词调优
```bash
vim ~/.config/fcitx5/conf/pinyin.conf
```
```ini
# 修改候选词数量
PageSize=9

# 调整词频权重（需安装中文词库）
vim /usr/share/fcitx5/addon/chinese-addons.conf
```
```lua
-- 在 Lua 脚本段修改排序算法：
sorter {
    name = "shuangpin_order"
    enable = true
}
```

## IBus/Fcitx 兼容性切换
```bash
# 临时切换
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx

# 永久切换（推荐）
im-config -n fcitx5 && reboot

# 恢复 IBus
sudo apt install --reinstall ibus
im-config -n ibus
```

> 验证当前输入法框架：
```bash
echo $XMODIFIERS
```
```text
@im=fcitx   # 显示当前生效框架
```

![输入法框架切换流程图](https://example.com/switch-diagram.png) 
（注：实际使用需替换有效流程图URL）
