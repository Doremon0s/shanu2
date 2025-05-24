import os
import boto3
import firebase_admin
from firebase_admin import credentials, db
from flask import Flask, request, jsonify
from werkzeug.utils import secure_filename
import datetime
from moviepy.editor import VideoFileClip
import tempfile
import uuid

# ✅ CONFIGURATION
WASABI_ACCESS_KEY = "VJY5L0G8IK7TSRJNEEFP"
WASABI_SECRET_KEY = "wXcO3XP7gDzOPjsriJQNISDSJCzoRpZNnibqO8IL"
WASABI_REGION = "us-east-1"
WASABI_BUCKET = "kistrontube"
WASABI_ENDPOINT = "https://s3.wasabisys.com"

FIREBASE_DATABASE_URL = "https://look-24b9a-default-rtdb.firebaseio.com/"
FIREBASE_CRED_PATH = "firebase_service_account.json"

# ✅ Firebase Init
cred = credentials.Certificate(FIREBASE_CRED_PATH)
firebase_admin.initialize_app(cred, {'databaseURL': FIREBASE_DATABASE_URL})
firebase_db = db.reference("feedItems")

# ✅ Wasabi Client Init
s3 = boto3.client("s3",
                  endpoint_url=WASABI_ENDPOINT,
                  aws_access_key_id=WASABI_ACCESS_KEY,
                  aws_secret_access_key=WASABI_SECRET_KEY,
                  region_name=WASABI_REGION)

# ✅ Flask App
app = Flask(__name__)

@app.route("/upload", methods=["POST"])
def upload_video():
    if "video" not in request.files:
        return jsonify({"error": "No video file provided"}), 400

    video = request.files["video"]
    filename = secure_filename(video.filename)
    file_id = str(uuid.uuid4())
    wasabi_filename = f"videos/{file_id}_{filename}"

    # Save to temp file
    with tempfile.NamedTemporaryFile(delete=False) as tmp:
        video.save(tmp.name)
        tmp_path = tmp.name

    # Upload to Wasabi
    s3.upload_file(tmp_path, WASABI_BUCKET, wasabi_filename, ExtraArgs={"ContentType": "video/mp4"})
    video_url = f"{WASABI_ENDPOINT}/{WASABI_BUCKET}/{wasabi_filename}"

    # Generate thumbnail & duration
    try:
        clip = VideoFileClip(tmp_path)
        duration = int(clip.duration)
        minutes = duration // 60
        seconds = duration % 60
        duration_str = f"{minutes}:{seconds:02d}"
    except Exception:
        duration = 0
        duration_str = "0:00"

    # Push to Firebase
    data = {
        "type": "video",
        "title": filename,
        "description": "",
        "channelName": "Uploader",
        "views": 0,
        "duration": duration_str,
        "durationInSeconds": duration,
        "thumbnailSrc": "https://via.placeholder.com/640x360/111/fff?text=Video",
        "fileUrl": video_url,
        "channelIconSrc": "https://via.placeholder.com/36/3ea6ff/fff?text=U",
        "category": "Video",
        "allCategories": ["Video"],
        "originalFileId": file_id,
        "createdAt": int(datetime.datetime.now().timestamp() * 1000),
        "videoType": "long",
        "source": "wasabi_upload"
    }

    firebase_db.push(data)
    return jsonify({"message": "✅ Video uploaded", "url": video_url})

@app.route("/")
def index():
    return "✅ Wasabi Upload Server Running"

# ✅ Main Run
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
