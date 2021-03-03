# puppeteer 동작을 영상으로 녹화 시키기

puppeteer 를 이용해서 작업을 진행하다 보면 진행사항을 보고 싶은 경우가 있다.

개발 중이거나 테스트시에는 직접 브라우저를 띄워서 볼수 있지만 실제 운영 환경에 적용을 하게 되면 직접 눈으로 불수 없기 때문에 답답한 상황이 계속 발생한다.

이런 답답함을 해결하기 위해 headless 상황에서 동작한 부분을 녹화하는 기능을 만들어 보았다.

## 기본적인 아이디어

1. 초당 N frame으로 스크린샷을 남기기
2. 스크린샷을 ffmpeg를 이용하여 영상으로 만들기

## Screenshot 남기기

### puppeteer의 page 객체에서 제공하는 screenshot API 사용

[Screenshot API](https://pptr.dev/#?product=Puppeteer&version=v1.19.0&show=api-pagescreenshotoptions)
```node
const puppeteer = require('puppeteer');

puppeteer.launch().then(async browser => {
  const page = await browser.newPage();
  await page.goto('https://example.com');
  await page.screenshot({path: 'screenshot.png'});
  await browser.close();
});
```

초당 10 Frame 남기기
```node
let count = 0;
let id = setInterval(() => {
    await page.screenshot({path: count + '.png'});
    count++;
}, 1000 / 10);

...

clearInterval(id);
```

puppeteer에서 제공해 주는 기본 API 를 사용하는 방법으로 초당 N Frame을 구현하려면 setInterval 로 동작중에 필요한 시점에 screenshot을 남기도록 구현을 해줘야 한다.

다만 이 방법으로 구현할시 스크린샷을 찍는 중간에 로직들이 비정상 동작하는  현상이 발생해서 다른 방법을 찾아봐야 했다.

### Chrome Dev-Tools의 screencast Method / Events 를 이용

[DevTools StartScreencast](https://chromedevtools.github.io/devtools-protocol/tot/Page/#method-startScreencast)

[DevTools screencastFrame](https://chromedevtools.github.io/devtools-protocol/tot/Page/#event-screencastFrame)
```node
const puppeteer = require('puppeteer');

puppeteer.launch().then(async browser => {
  const page = await browser.newPage();
  const client = await page.target().createCDPSession();
  await client.send('Page.startScreencast', {format: 'png', everyNthFrame: 1, quality: 100});
  await client.on('Page.screencastFrame', async (args) => {
    fs.writeFileSync(args.metadata.timestamp + 'screenshot.png', Buffer.from(args.data, 'base64'));
  });
  await page.goto('https://example.com');
  await client.send('Page.stopScreencast');
  await browser.close();
});
```

chrome Dev tools 를 가져와서 브라우저에서 자체적으로 제공하는 screencast 기능을 활용하여 screenshot을 남기게 구현한다.

따로 interval을 걸 필요가 없지만 순서가 보장되지 않는 단점이 있어서 완벽한 순서 보장을 하려면 meta로 넘어오는 timestamp 으로 파일명을 만들어 준다.

## ffmpeg를 가지고 영상 만들기

[FFMPEG Doc](https://www.ffmpeg.org/ffmpeg.html)

ffmpeg의 기능을 가지고 시간순으로 생성한 screenshot*.png 들을 영상으로 만들어 준다.

```sh
ffmpeg -pattern_type glob -i "*.png" -pix_fmt yuv420p -deinterlace -vf "scale=640:360" -vsync 1 -threads 0 -vcodec libx264 -g 60 -sc_threshold 0 -b:v 1024k -bufsize 1216k -maxrate 1280k -preset medium -profile:v main -tune film -f mp4 -y result.mp4
```

## 간단한 샘플 코드 구현

[4ppl3Hun73r/puppeteer-rec-video](https://github.com/4ppl3Hun73r/puppeteer-rec-video)