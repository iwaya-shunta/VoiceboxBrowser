import os
import sys
import subprocess
import requests
from openai import OpenAI

# --- 定数設定 ---
# OpenAI APIのモデル
OPENAI_MODEL = "gpt-4o" 
# レビューを依頼する際のシステムプロンプト
SYSTEM_PROMPT = """
あなたは、世界トップクラスのソフトウェアエンジニアであり、コードレビュアーです。
提供されたコードのdiff情報から、以下の観点でレビューを行ってください。

- **堅牢性**: バグや潜在的な問題を指摘してください。
- **可読性**: コードの読みやすさやメンテナンス性を向上させる提案をしてください。
- **パフォーマンス**: パフォーマンスに影響を与える可能性のある箇所を指摘してください。
- **セキュリティ**: 脆弱性やセキュリティリスクがないか確認してください。
- **ベストプラクティス**: 一般的なコーディング規約や設計原則に従っているか確認してください。

レビューコメントは簡潔かつ具体的に、日本語で記述してください。
指摘事項がない場合は、「特に指摘事項はありません。」とだけ返答してください。
"""
# トークン数の上限（コスト管理とAPIエラー防止のため）
MAX_TOKENS_FOR_DIFF = 4000 

# --- 環境変数から情報を取得 ---
try:
    github_token = os.environ['GITHUB_TOKEN']
    openai_api_key = os.environ['OPENAI_API_KEY']
    pr_number = os.environ['PR_NUMBER']
    repo_name = os.environ['GITHUB_REPOSITORY'] # 'owner/repo'の形式
    base_sha = os.environ['BASE_SHA']
    head_sha = os.environ['HEAD_SHA']
except KeyError as e:
    print(f"エラー: 環境変数 {e} が設定されていません。")
    sys.exit(1)

# --- 1. Gitの差分を取得 ---
def get_git_diff():
    try:
        # PRのベースブランチとヘッドブランチ間の差分を取得
        command = ["git", "diff", f"{base_sha}...{head_sha}"]
        result = subprocess.run(command, capture_output=True, text=True, check=True)
        return result.stdout
    except subprocess.CalledProcessError as e:
        print(f"Git diffの取得に失敗しました: {e.stderr}")
        return None

# --- 2. OpenAI APIにレビューを依頼 ---
def get_ai_review(diff):
    if not diff:
        return "コードの差分が取得できませんでした。"
    
    # 差分が長すぎる場合は処理を中断
    if len(diff) > MAX_TOKENS_FOR_DIFF:
        return f"コード差分が長すぎるため({len(diff)}文字)、レビューをスキップしました。ファイルを分割するなどの対応をご検討ください。"

    try:
        client = OpenAI(api_key=openai_api_key)
        response = client.chat.completions.create(
            model=OPENAI_MODEL,
            messages=[
                {"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user", "content": f"以下のコード差分をレビューしてください:\n\n```diff\n{diff}\n```"}
            ],
            temperature=0.5, # 創造性を抑え、事実に基づいた回答を促す
        )
        return response.choices[0].message.content
    except Exception as e:
        print(f"OpenAI APIの呼び出しでエラーが発生しました: {e}")
        return f"AIレビューの生成中にエラーが発生しました: {e}"

# --- 3. GitHubのPRにコメントを投稿 ---
def post_comment_to_pr(comment):
    url = f"https://api.github.com/repos/{repo_name}/issues/{pr_number}/comments"
    headers = {
        "Authorization": f"token {github_token}",
        "Accept": "application/vnd.github.v3+json",
    }
    data = {"body": f"### AIによる自動コードレビュー\n\n{comment}"}
    
    response = requests.post(url, headers=headers, json=data)
    
    if response.status_code == 201:
        print("コメントの投稿に成功しました。")
    else:
        print(f"コメントの投稿に失敗しました。ステータスコード: {response.status_code}, レスポンス: {response.text}")
        sys.exit(1)

# --- メイン処理 ---
if __name__ == "__main__":
    print("1. コード差分の取得を開始...")
    diff_text = get_git_diff()
    
    if diff_text:
        print("2. AIによるレビューを生成中...")
        review_comment = get_ai_review(diff_text)
        
        if review_comment:
            print("3. GitHubにレビューコメントを投稿中...")
            post_comment_to_pr(review_comment)
        else:
            print("AIからのレビューが空でした。")
    else:
        print("差分がないため、処理を終了します。")

    print("処理完了。")
