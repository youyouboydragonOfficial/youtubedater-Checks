<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>YouTube 動画情報取得</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600&display=swap');
        body {
            font-family: 'Inter', sans-serif;
        }
        .spinner {
            border: 4px solid rgba(0, 0, 0, 0.1);
            width: 36px;
            height: 36px;
            border-radius: 50%;
            border-left-color: #3b82f6;
            animation: spin 1s ease infinite;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
    </style>
</head>
<body class="bg-gray-100 flex items-center justify-center min-h-screen p-4">

    <div class="bg-white p-8 rounded-2xl shadow-xl max-w-lg w-full text-center">
        <h1 class="text-3xl font-bold mb-4 text-gray-800">YouTube 動画情報取得</h1>
        <p class="text-gray-600 mb-6">動画のURLを入力してください。</p>
        
        <form id="infoForm" class="space-y-4">
            <input 
                type="text" 
                id="videoUrl" 
                placeholder="YouTubeの動画URLをここに貼り付け"
                class="w-full px-4 py-3 rounded-xl border border-gray-300 focus:outline-none focus:ring-2 focus:ring-blue-500 transition-all duration-200"
            />
            <button 
                type="submit" 
                id="infoButton"
                class="w-full bg-blue-600 text-white font-semibold py-3 px-4 rounded-xl shadow-lg hover:bg-blue-700 transition-all duration-200 transform hover:scale-105 flex items-center justify-center"
            >
                <span id="buttonText">情報を取得</span>
                <div id="loadingSpinner" class="spinner hidden ml-2"></div>
            </button>
        </form>

        <div id="messageArea" class="mt-6 text-left">
            <!-- メッセージがここに表示されます -->
        </div>

        <div id="videoInfo" class="mt-6 text-left hidden">
            <h2 class="text-2xl font-semibold mb-4 text-gray-800">動画詳細</h2>
            <img id="thumbnail" class="w-full h-auto rounded-lg mb-4 shadow-md" alt="動画サムネイル">
            <p class="text-lg font-bold">タイトル:</p>
            <p id="title" class="mb-2 text-gray-700"></p>
            <p class="text-lg font-bold">再生時間:</p>
            <p id="duration" class="mb-2 text-gray-700"></p>
            <p class="text-lg font-bold">画質:</p>
            <p id="quality" class="mb-2 text-gray-700"></p>
        </div>
    </div>

    <script>
        document.addEventListener('DOMContentLoaded', () => {
            const form = document.getElementById('infoForm');
            const videoUrlInput = document.getElementById('videoUrl');
            const messageArea = document.getElementById('messageArea');
            const infoButton = document.getElementById('infoButton');
            const buttonText = document.getElementById('buttonText');
            const loadingSpinner = document.getElementById('loadingSpinner');
            const videoInfoDiv = document.getElementById('videoInfo');
            const thumbnailImg = document.getElementById('thumbnail');
            const titleP = document.getElementById('title');
            const durationP = document.getElementById('duration');
            const qualityP = document.getElementById('quality');

            form.addEventListener('submit', async (e) => {
                e.preventDefault();

                const url = videoUrlInput.value.trim();
                videoInfoDiv.classList.add('hidden');
                showMessage('info', '動画情報を取得しています...');
                infoButton.disabled = true;
                buttonText.textContent = '取得中...';
                loadingSpinner.classList.remove('hidden');

                try {
                    // YouTube APIを使わずに直接HTMLをパースする簡単な方法
                    // これは非常に不安定な方法であり、YouTubeのHTML構造変更で動かなくなる可能性があります。
                    // 信頼性の高いアプリケーションには適していません。
                    const proxyUrl = `https://api.allorigins.win/get?url=${encodeURIComponent(url)}`;
                    const response = await fetch(proxyUrl);
                    const jsonResponse = await response.json();
                    const text = jsonResponse.contents;
                    
                    const doc = new DOMParser().parseFromString(text, 'text/html');
                    
                    // エラーを修正し、`ytInitialPlayerResponse`を含むスクリプトタグを見つける
                    const scriptTag = Array.from(doc.querySelectorAll('script')).find(script => 
                        script.textContent.includes('ytInitialPlayerResponse')
                    );

                    if (!scriptTag) {
                        throw new Error('動画情報が見つかりません。URLを確認してください。');
                    }
                    
                    // 正規表現をより柔軟に修正
                    const jsonMatch = scriptTag.textContent.match(/ytInitialPlayerResponse\s*=\s*({[\s\S]*?})\s*;/);

                    if (!jsonMatch || !jsonMatch[1]) {
                        throw new Error('動画データの解析に失敗しました。');
                    }

                    const jsonString = jsonMatch[1];
                    const data = JSON.parse(jsonString);

                    const videoDetails = data.videoDetails;

                    if (!videoDetails) {
                        throw new Error('動画情報が取得できませんでした。');
                    }
                    
                    // 再生時間を秒単位から分:秒形式に変換
                    const seconds = parseInt(videoDetails.lengthSeconds, 10);
                    const minutes = Math.floor(seconds / 60);
                    const remainingSeconds = seconds % 60;
                    const formattedDuration = `${minutes}:${remainingSeconds < 10 ? '0' : ''}${remainingSeconds}`;
                    
                    // 画質情報は利用可能な最高画質を取得
                    const quality = data.streamingData.formats
                        .sort((a, b) => b.height - a.height)
                        .find(f => f.qualityLabel)
                        ?.qualityLabel || '不明';

                    thumbnailImg.src = videoDetails.thumbnail.thumbnails[0].url;
                    titleP.textContent = videoDetails.title;
                    durationP.textContent = formattedDuration;
                    qualityP.textContent = quality;
                    
                    videoInfoDiv.classList.remove('hidden');
                    showMessage('success', '情報の取得が完了しました。');

                } catch (error) {
                    console.error('Fetch error:', error);
                    showMessage('error', `エラーが発生しました: ${error.message}`);
                } finally {
                    infoButton.disabled = false;
                    buttonText.textContent = '情報を取得';
                    loadingSpinner.classList.add('hidden');
                }
            });

            function showMessage(type, text) {
                let colorClass, title;
                switch (type) {
                    case 'info':
                        colorClass = 'bg-blue-100 border-blue-400 text-blue-700';
                        title = '情報:';
                        break;
                    case 'error':
                        colorClass = 'bg-red-100 border-red-400 text-red-700';
                        title = 'エラー:';
                        break;
                    case 'success':
                        colorClass = 'bg-green-100 border-green-400 text-green-700';
                        title = '成功:';
                        break;
                    default:
                        colorClass = 'bg-gray-100 border-gray-400 text-gray-700';
                        title = 'メッセージ:';
                }

                const message = `
                    <div class="p-4 rounded-lg relative shadow-md ${colorClass}" role="alert">
                        <strong class="font-bold">${title}</strong>
                        <span class="block sm:inline">${text}</span>
                    </div>
                `;
                messageArea.innerHTML = message;
            }
        });
    </script>
</body>
</html>
