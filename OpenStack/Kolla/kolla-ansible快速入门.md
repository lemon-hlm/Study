
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [1 概述](#1-概述)
* [2 kolla\-ansible命令](#2-kolla-ansible命令)

<!-- /code_chunk_output -->

# 1 概述

kolla\-ansible是一个结构相对简单的项目，它通过一个shell脚本，根据用户的参数，选择不同的playbook和不同的参数调用ansible-playbook执行，没有数据库，没有消息队列，所以本文的重点是ansible本身的语法。

# 2 kolla\-ansible命令

查看kolla-ansible/tools/kolla-ansible文件内容, 主要命令代码如下:

```sh
function find_base_dir {
    # 略
}

function process_cmd {
    echo "$ACTION : $CMD"
    $CMD
    if [[ $? -ne 0 ]]; then
        echo "Command failed $CMD"
        exit 1
    fi
}

function usage {
cat <<EOF
# 略   
EOF
}

function bash_completion {
cat <<EOF
# 略  
EOF
}

INVENTORY="${BASEDIR}/ansible/inventory/all-in-one"
PLAYBOOK="${BASEDIR}/ansible/site.yml"
VERBOSITY=
EXTRA_OPTS=${EXTRA_OPTS}
CONFIG_DIR="/etc/kolla"
PASSWORDS_FILE="${CONFIG_DIR}/passwords.yml"
DANGER_CONFIRM=
INCLUDE_IMAGES=
INCLUDE_DEV=
BACKUP_TYPE="full"

while [ "$#" -gt 0 ]; do
    case "$1" in

    (--inventory|-i)
            INVENTORY="$2"
            shift 2
            ;;
# 支持的各种参数, 略
esac
done

case "$1" in

(prechecks)
        ACTION="Pre-deployment checking"
        EXTRA_OPTS="$EXTRA_OPTS -e kolla_action=precheck"
        ;;
(check)
        ACTION="Post-deployment checking"
        EXTRA_OPTS="$EXTRA_OPTS -e kolla_action=check"
        ;;
# 以下忽略
esac

CONFIG_OPTS="-e @${CONFIG_DIR}/globals.yml -e @${PASSWORDS_FILE} -e CONFIG_DIR=${CONFIG_DIR}"
CMD="ansible-playbook -i $INVENTORY $CONFIG_OPTS $EXTRA_OPTS $PLAYBOOK $VERBOSITY"
process_cmd
```

可以看到, 执行当我们执行kolla\-ansible deploy时，kolla-ansible命令帮我们调用了对应的ansible\-playbook执行，除此之外，没有其他工作。所以对于kolla-ansible项目，主要学习ansible语法即可。