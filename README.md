# Sobre
Módulo feito com base no pacote [nodejs-meme-generator.](https://github.com/TehZarathustra/nodejs-meme-generator#readme "Autor: TehZarathustra")

## Exemplo de uso
```
const MemeLib = require('nodejs-meme-generator');
const sizeOf = require('buffer-image-size'); // Opcional | detecta as dimensões da foto automaticamente
const fs = require('fs');
const twit = require('twit');


const bot = new twit({
  consumer_key: 'consumerKey',
  consumer_secret: 'consumerSecret',
  access_token: 'accessToken',
  access_token_secret: 'accessTokenSecret',
});

const stream = bot.stream('statuses/filter', {
  track: 'Tag #bot ou Marcação @bot',
  tweet_mode: 'extended',
});

stream.on('tweet', async (tweet) => {
  try {
    const tweetID = tweet.id_str;
    const userName = tweet.user.screen_name;
    const {
      truncated,
    } = tweet;
    // Recebe o texto do tweet de acordo com seu tipo: compat | extended;
    const memeText = truncated ? tweet.extended_tweet.full_text : tweet.text;

    fs.readFile(`${__dirname}/caminhoDafoto`, (err, imageData) => {
      if (err) throw err;
      const dimensions = sizeOf(imageData);

      const memeGenerator = new MemeLib({
        canvasOptions: {
          canvasWidth: dimensions.width, // Opcional | valor padrão = 500
          canvasHeight: dimensions.height, // Opcional | valor padrão = 500
        },
        /*
        Caso não use o módulo buffer-image-size, verifique as dimensões da foto!
        canvasOptions: {
          canvasWidth: largura, // Opcional | valor padrão = 500
          canvasHeight: altura, // Opcional | valor padrão = 500
        }, */
        fontOptions: {
          fontSize: 32, // Opcional | valor padrão = 46
          fontFamily: 'Comic Sans', // Opcional | valor padrão = impact
          lineHeight: 2, // Opcional | valor padrão = 2
        },
      });

      memeGenerator.generateMeme({
        bottomText: memeText,
        url: imageData,
      })
        .then(async (meme) => {
          // Remove o header base64 antes de fazer o upload;
          const memeData = meme.replace('data:image/png;base64,', '');
          await bot.post('media/upload', {
            media_data: memeData,
            media_category: 'tweet_image',
          }).then(async ({
            data,
          }) => {
            const mediaIdStr = data.media_id_string;
            const altText = memeText;
            const metaParams = {
              media_id: mediaIdStr,
              alt_text: {
                text: altText,
              },
            };

            await bot.post('media/metadata/create', metaParams)
              .catch((error) => console.log(error))
              .then(() => {
                bot.post('statuses/update', {
                  in_reply_to_status_id: tweetID,
                  status: `@${userName}`,
                  media_ids: [mediaIdStr],
                }).catch((error) => console.log(`Erro ao fazer o upload: ${error}`))
                  .then(
                    () => {
                      console.log(`Resposta enviada para @${userName}`);
                    },
                  );
              });
          });
        });
    });
  } catch (f) {
    console.log(`Erro na streaming API: ${f}`);
  }
});
```
