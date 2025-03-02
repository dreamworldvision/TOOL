// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const stripe = require('stripe')('your-stripe-secret-key');
const ffmpeg = require('fluent-ffmpeg');
const fs = require('fs');
const { generateVoice } = require('./textToSpeech');
const User = require('./models/User');

const app = express();
app.use(express.json());
app.use(cors());

mongoose.connect('mongodb://localhost:27017/textToVideo', {
  useNewUrlParser: true,
  useUnifiedTopology: true,
});

app.post('/generate-video', async (req, res) => {
  const { text, userId } = req.body;
  
  if (!text || !userId) return res.status(400).json({ error: 'Missing data' });

  const audioPath = `./output/${userId}.mp3`;
  const videoPath = `./output/${userId}.mp4`;

  await generateVoice(text, audioPath);

  ffmpeg()
    .input(audioPath)
    .input('./background.jpg') // Use a static image
    .output(videoPath)
    .on('end', () => res.json({ videoUrl: videoPath }))
    .run();
});

app.post('/payment', async (req, res) => {
  const { userId, amount } = req.body;

  const session = await stripe.checkout.sessions.create({
    payment_method_types: ['card'],
    line_items: [{
      price_data: {
        currency: 'usd',
        product_data: { name: 'Text-to-Video Generation' },
        unit_amount: amount * 100,
      },
      quantity: 1,
    }],
    mode: 'payment',
    success_url: 'http://localhost:3000/success',
    cancel_url: 'http://localhost:3000/cancel',
  });

  await User.findByIdAndUpdate(userId, { $inc: { credits: amount } });
  res.json({ url: session.url });
});

app.listen(5000, () => console.log('Server running on port 5000'));
