# FMDataMigrationWrap

## コマンドの実行例

```sh
teruhiro@MacBook-Pro-13:~/tmp/FMDataMigrationWrap/example $ bash FMDataMigrationWrap.sh -src_account "admin" -src_pwd "admin" -clone_account "admin" -clone_pwd "admin" -reevaluate -rebuildindexes
```

## FMDataMigrationWrap.sh

```sh
#!/usr/bin/env sh
set -euo pipefail

##################################################
# history
##################################################
# 2021-04-02 created komaki@frudens.jp

##################################################
# GLOBAL
##################################################
MIGRATION_PARAM="$@"
PATH_FMDM=
PROD_PATH= PROD_LIST= PROD_COUNT=0
CLONE_PATH= CLONE_LIST= CLONE_COUNT=0
TARGET_PATH=
RESULT=

##################################################
# function
##################################################
func_log() {
  local path_log="./FMDataMigrationWrap_log.txt"
  local param="$@"
  local ts=$(date +"%Y-%m-%dT%H:%M:%S")
  echo "$ts $param" | tee -a "$path_log"
}

func_create_target_dir() {
  local target_path=$(printf "$1" | sed -e 's/prod/migrated/')
  local target_dir=$(dirname "$target_path")
  if [ ! -d "$target_dir" ]; then
    mkdir -p "$target_dir"
  fi
  echo "$target_path"
}

func_find_clone_path() {
  local fileName_ext=$(basename "$1")
  local fileName="${fileName_ext%.*}"
  local clone_path=$(printf "$CLONE_LIST" | grep "$fileName")
  echo "$clone_path"
}

##################################################
# start FMDataMigrationWrap.sh
##################################################
func_log "=============================="
func_log "FMDataMigrationWrap.shを開始しました。"

##################################################
# check FMDataMigration
##################################################
if type "FMDataMigration" >/dev/null 2>&1; then
  PATH_FMDM="FMDataMigration"
  func_log "FMDataMigrationが見つかりました。"
else
  if [ -e "./FMDataMigration" ]; then
    PATH_FMDM="./FMDataMigration"
    func_log "FMDataMigrationが見つかりました。"
  else
    func_log "FMDataMigrationが見つかりませんでした。"
    exit 1
  fi
fi

##################################################
# check ./resources/prod folder
##################################################
if [ -d "./resources/prod" ]; then
  PROD_LIST=$(find ./resources/prod -type f -name '*.fmp12')
  PROD_COUNT=$(echo $(echo "$PROD_LIST" | wc -l))
fi

##################################################
# check ./resources/clone folder
##################################################
if [ -d "./resources/clone" ]; then
  CLONE_LIST=$(find ./resources/clone -type f -name '*.fmp12')
  CLONE_COUNT=$(echo $(echo "$CLONE_LIST" | wc -l))
fi

##################################################
# check files
##################################################
if [ ! $CLONE_COUNT -ge 1 ]; then
  func_log "cloneフォルダがないかfmp12ファイルが1つもないため終了します。"
  exit 1
fi

if [ $PROD_COUNT -ne $CLONE_COUNT ]; then
  func_log "prodフォルダとcloneフォルダのファイル数が一致していないため終了します。"
  exit 1
fi

# for i in $PROD_LIST; do
#   TMP_PROD_PATH="$i"
#   TMP_FILENAME_EXT=$(basename "$TMP_PROD_PATH")
#   TMP_FILENAME="${TMP_FILENAME_EXT%.*}"
#   TMP_CLONE_FOUND=$(echo $(echo "$CLONE_LIST" | grep "$TMP_FILENAME" | wc -l))
#   if [ $TMP_CLONE_FOUND -ne 1 ]; then
#     func_log "cloneフォルダのなかに [ $TMP_FILENAME ] にマッチするファイルが複数あります。"
#     exit 1
#   fi
# done

##################################################
# run
##################################################
func_log 対象ファイル数: "$PROD_COUNT"
func_log 対象ファイル: $(echo -n "$PROD_LIST" | tr '\n' ', ')

for i in $PROD_LIST; do
  PROD_PATH="$i"
  func_log "------------------------------"
  func_log "$PROD_PATH" をマイグレーションします。
  func_log "------------------------------"
  CLONE_PATH=$(func_find_clone_path "$PROD_PATH")
  TARGET_PATH=$(func_create_target_dir "$PROD_PATH")
  RESULT=$("$PATH_FMDM" -src_path "$PROD_PATH" -clone_path "$CLONE_PATH" -target_path "$TARGET_PATH" $MIGRATION_PARAM)
  func_log "$RESULT"
done
```
