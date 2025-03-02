import os
import argparse
from dotenv import load_dotenv
from gtts import gTTS
from moviepy.editor import *
from PIL import Image
import requests
import numpy as np

# Load environment variables
load_dotenv()
STABILITY_API_KEY = os.getenv("STABILITY_API_KEY")

def generate_image(prompt, output_path="temp_image.png"):
    """Generate image using Stability AI API"""
    headers = {
        "Authorization": f"Bearer {STABILITY_API_KEY}",
        "Content-Type": "application/json"
    }

    data = {
        "text_prompts": [{"text": prompt}],
        "height": 512,
        "width": 512,
        "steps": 30
    }

    response = requests.post(
        "https://api.stability.ai/v1/generation/stable-diffusion-512-v2-1/text-to-image",
        headers=headers,
        json=data
    )

    if response.status_code == 200:
        with open(output_path, "wb") as f:
            f.write(response.content)
        return output_path
    else:
        raise Exception(f"Image generation failed: {response.text}")

def text_to_speech(text, output_path="temp_audio.mp3"):
    """Convert text to speech using gTTS"""
    tts = gTTS(text=text, lang='en', slow=False)
    tts.save(output_path)
    return output_path

def create_video_clip(image_path, audio_path, duration):
    """Create video clip with image and audio"""
    image = ImageClip(image_path).set_duration(duration)
    audio = AudioFileClip(audio_path)
    return image.set_audio(audio)

def text_to_video(text, output_file="output.mp4"):
    """Main conversion function"""
    # Split text into sentences
    sentences = [s.strip() for s in text.split('.') if s.strip()]
    
    clips = []
    
    for sentence in sentences:
        try:
            # Generate assets for each sentence
            img_path = generate_image(sentence)
            audio_path = text_to_speech(sentence)
            
            # Get audio duration
            audio = AudioFileClip(audio_path)
            duration = audio.duration
            
            # Create clip
            clip = create_video_clip(img_path, audio_path, duration)
            clips.append(clip)
        except Exception as e:
            print(f"Error processing sentence '{sentence}': {str(e)}")
    
    # Combine all clips
    final_video = concatenate_videoclips(clips)
    final_video.write_videofile(output_file, fps=24)
    
    # Cleanup temp files
    for f in ["temp_image.png", "temp_audio.mp3"]:
        if os.path.exists(f):
            os.remove(f)
    
    return output_file

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Convert text to video")
    parser.add_argument("--text", type=str, required=True, help="Input text")
    parser.add_argument("--output", type=str, default="output.mp4", help="Output video file")
    args = parser.parse_args()
    
    result = text_to_video(args.text, args.output)
    print(f"Video created: {result}")
