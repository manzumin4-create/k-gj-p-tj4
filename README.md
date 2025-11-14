<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>文才向上アシスタント</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body { font-family: 'Inter', sans-serif; background-color: #f7f7f7; }
        .container { max-width: 900px; }
        .glow-button {
            transition: all 0.3s ease;
            box-shadow: 0 4px 15px 0 rgba(100, 100, 255, 0.5);
        }
        .glow-button:hover {
            box-shadow: 0 4px 20px 0 rgba(100, 100, 255, 0.8);
            transform: translateY(-1px);
        }
    </style>
</head>
<body class="p-4 sm:p-8">

    <div class="container mx-auto">
        <h1 class="text-3xl font-bold mb-6 text-gray-800 border-b pb-2">小説文才向上アシスタント</h1>
        <p class="mb-6 text-gray-600">あなたの書いた二次創作の文章を入力してください。AIが「文才がある」と感じられるような、豊かで魅力的な描写や表現を提案・修正します。</p>

        <!-- 入力エリア -->
        <div class="bg-white p-6 rounded-xl shadow-lg mb-6">
            <label for="draftText" class="block text-lg font-semibold mb-2 text-gray-700">あなたの文章（添削してほしい原稿）</label>
            <textarea id="draftText" rows="8" class="w-full p-4 border-2 border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500 transition duration-150" placeholder="例：彼女は部屋に入った。彼はそこに立っていた。私たちは何も言わなかった。">{{ 例の文章 }}</textarea>
            <p class="text-sm mt-2 text-gray-500">（{{ 例の文章の説明 }}）</p>
        </div>

        <!-- ボタン -->
        <button id="polishButton" class="w-full px-6 py-3 bg-indigo-600 text-white font-bold rounded-xl text-lg glow-button hover:bg-indigo-700 transition duration-300 flex items-center justify-center disabled:opacity-50" onclick="requestPolish()" disabled>
            <span id="buttonText">文章を添削・描写追加</span>
            <svg id="loadingSpinner" class="animate-spin -ml-1 mr-3 h-5 w-5 text-white hidden" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
            </svg>
        </button>

        <!-- 結果表示エリア -->
        <div class="mt-8 bg-white p-6 rounded-xl shadow-lg border-t-4 border-indigo-500">
            <h2 class="text-xl font-semibold mb-4 text-gray-700">AIによる提案・添削後の文章</h2>
            <div id="resultOutput" class="min-h-[150px] p-4 bg-gray-50 border border-gray-200 rounded-lg whitespace-pre-wrap text-gray-800 leading-relaxed">
                ここにAIが添削した文章が表示されます。
            </div>
        </div>

    </div>

    <script>
        // グローバル変数
        const DRAFT_EXAMPLE = "彼女は部屋に入った。彼はそこに立っていた。私たちは何も言わなかった。重い沈黙だけが、二人を分かつ壁だった。"
        const DRAFT_EXAMPLE_DESC = "（この例文は、AIがより叙情的・文学的な表現に修正する例です。例としてそのまま使用することも、消して独自の文章を入力することもできます。）"
        
        const MODEL_NAME = "gemini-2.5-flash-preview-09-2025";
        const API_KEY = ""; // Canvas環境で自動で提供されるため、空のままにします
        const API_URL = `https://generativelanguage.googleapis.com/v1beta/models/${MODEL_NAME}:generateContent?key=${API_KEY}`;

        // DOM要素
        const draftTextarea = document.getElementById('draftText');
        const polishButton = document.getElementById('polishButton');
        const resultOutput = document.getElementById('resultOutput');
        const buttonText = document.getElementById('buttonText');
        const loadingSpinner = document.getElementById('loadingSpinner');

        // 初期設定
        draftTextarea.value = DRAFT_EXAMPLE;

        /**
         * 処理中のUIを更新
         * @param {boolean} isLoading - ロード中かどうか
         */
        function setLoading(isLoading) {
            polishButton.disabled = isLoading;
            if (isLoading) {
                loadingSpinner.classList.remove('hidden');
                buttonText.textContent = '添削中...';
                polishButton.classList.add('opacity-50');
            } else {
                loadingSpinner.classList.add('hidden');
                buttonText.textContent = '文章を添削・描写追加';
                polishButton.classList.remove('opacity-50');
            }
        }

        /**
         * Gemini APIへのリクエストをexponential backoff付きで実行
         * @param {Object} payload - APIリクエストのペイロード
         * @param {number} maxRetries - 最大リトライ回数
         * @returns {Promise<Object>} APIレスポンスのJSON
         */
        async function fetchWithBackoff(payload, maxRetries = 5) {
            for (let attempt = 0; attempt < maxRetries; attempt++) {
                try {
                    const response = await fetch(API_URL, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify(payload)
                    });

                    if (response.ok) {
                        return await response.json();
                    } else if (response.status === 429 && attempt < maxRetries - 1) {
                        // Rate limit exceeded (429), wait and retry
                        const delay = Math.pow(2, attempt) * 1000 + Math.random() * 1000;
                        await new Promise(resolve => setTimeout(resolve, delay));
                        // console.log(`Retrying API call. Attempt ${attempt + 2}/${maxRetries}`);
                        continue;
                    } else {
                        throw new Error(`API Error: ${response.status} ${response.statusText}`);
                    }
                } catch (error) {
                    if (attempt < maxRetries - 1) {
                        const delay = Math.pow(2, attempt) * 1000 + Math.random() * 1000;
                        await new Promise(resolve => setTimeout(resolve, delay));
                        // console.log(`Retry due to network/fetch error. Attempt ${attempt + 2}/${maxRetries}`);
                        continue;
                    } else {
                        throw error;
                    }
                }
            }
            throw new Error('API call failed after multiple retries.');
        }


        /**
         * ユーザーの文章の添削をリクエスト
         */
        async function requestPolish() {
            const userText = draftTextarea.value.trim();
            if (!userText) {
                resultOutput.textContent = '添削する文章を入力してください。';
                return;
            }

            setLoading(true);
            resultOutput.textContent = 'AIが最高の表現を探しています... しばらくお待ちください。';

            // プロの小説家/編集者として振る舞うシステム命令
            const systemPrompt = `あなたはプロの小説家・編集者であり、特に二次創作の文体向上に特化しています。
以下のユーザーの文章を読み、それを「文才がある」「叙情的で引き込まれる」と感じられるように、以下を重点的にブラッシュアップしてください。

1. **描写の強化**: 五感に訴えかける具体的な表現（色、音、匂い、触感、感情の機微）を追加し、情景やキャラクターの心情を鮮やかに描き出してください。
2. **表現の洗練**: 平凡な表現を、比喩、擬人化、倒置法などの修辞技法を用いて文学的に洗練させ、文章にリズムと深みを与えてください。
3. **文体の調整**: 全体のトーン（雰囲気）を維持しつつ、より感情の機微が伝わるような語彙を選択してください。

出力は、**修正後の文章本体のみ**を提示し、説明やコメントは一切付け加えないでください。`;

            const userQuery = `以下の文章を、文才があると感じられるように、描写を加えてより洗練された表現に修正してください。

---
${userText}
---`;

            const payload = {
                contents: [{ parts: [{ text: userQuery }] }],
                systemInstruction: {
                    parts: [{ text: systemPrompt }]
                },
                // Google Searchは不要
            };

            try {
                const result = await fetchWithBackoff(payload);
                const candidate = result.candidates?.[0];

                if (candidate && candidate.content?.parts?.[0]?.text) {
                    const polishedText = candidate.content.parts[0].text;
                    resultOutput.textContent = polishedText;
                } else {
                    resultOutput.textContent = 'AIからの応答が取得できませんでした。時間をおいて再度お試しください。';
                }
            } catch (error) {
                console.error("API Request Failed:", error);
                resultOutput.textContent = `エラーが発生しました: ${error.message}。ネットワーク接続またはAPIの制限を確認してください。`;
            } finally {
                setLoading(false);
            }
        }

        // 初期状態でボタンを有効化
        document.addEventListener('DOMContentLoaded', () => {
            polishButton.disabled = false;
        });
    </script>
</body>
</html>


