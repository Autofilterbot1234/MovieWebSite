import os
import sys
import re
import requests
from flask import Flask, render_template_string, request, redirect, url_for, Response, jsonify
from pymongo import MongoClient
from bson.objectid import ObjectId
from functools import wraps
from datetime import datetime, timedelta
from apscheduler.schedulers.background import BackgroundScheduler

# ======================================================================
# --- আপনার ব্যক্তিগত ও অ্যাডমিন তথ্য (এনভায়রনমেন্ট থেকে লোড হবে) ---
# ======================================================================
MONGO_URI = os.environ.get("MONGO_URI")
BOT_TOKEN = os.environ.get("BOT_TOKEN")
TMDB_API_KEY = os.environ.get("TMDB_API_KEY")
ADMIN_CHANNEL_ID = os.environ.get("ADMIN_CHANNEL_ID")
BOT_USERNAME = os.environ.get("BOT_USERNAME")
ADMIN_USERNAME = os.environ.get("ADMIN_USERNAME")
ADMIN_PASSWORD = os.environ.get("ADMIN_PASSWORD")

# আপনার চ্যানেল এবং ডেভেলপারের তথ্য
MAIN_CHANNEL_LINK = os.environ.get("MAIN_CHANNEL_LINK")
UPDATE_CHANNEL_LINK = os.environ.get("UPDATE_CHANNEL_LINK")
DEVELOPER_USER_LINK = os.environ.get("DEVELOPER_USER_LINK")

# --- প্রয়োজনীয় ভেরিয়েবলগুলো সেট করা হয়েছে কিনা তা পরীক্ষা করা ---
required_vars = {
    "MONGO_URI": MONGO_URI, "BOT_TOKEN": BOT_TOKEN, "TMDB_API_KEY": TMDB_API_KEY,
    "ADMIN_CHANNEL_ID": ADMIN_CHANNEL_ID, "BOT_USERNAME": BOT_USERNAME,
    "ADMIN_USERNAME": ADMIN_USERNAME, "ADMIN_PASSWORD": ADMIN_PASSWORD,
    "MAIN_CHANNEL_LINK": MAIN_CHANNEL_LINK,
    "UPDATE_CHANNEL_LINK": UPDATE_CHANNEL_LINK,
    "DEVELOPER_USER_LINK": DEVELOPER_USER_LINK,
}

missing_vars = [name for name, value in required_vars.items() if not value]
if missing_vars:
    print(f"FATAL: Missing required environment variables: {', '.join(missing_vars)}")
    print("Please set these variables in your deployment environment and restart the application.")
    sys.exit(1)

# ======================================================================
# --- অ্যাপ্লিকেশন সেটআপ এবং অন্যান্য ফাংশন ---
# ======================================================================
TELEGRAM_API_URL = f"https://api.telegram.org/bot{BOT_TOKEN}"
app = Flask(__name__)

def check_auth(username, password):
    return username == ADMIN_USERNAME and password == ADMIN_PASSWORD

def authenticate():
    return Response('Could not verify your access level.', 401, {'WWW-Authenticate': 'Basic realm="Login Required"'})

def requires_auth(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        auth = request.authorization
        if not auth or not check_auth(auth.username, auth.password):
            return authenticate()
        return f(*args, **kwargs)
    return decorated

try:
    client = MongoClient(MONGO_URI)
    db = client["movie_db"]
    movies = db["movies"]
    settings = db["settings"]
    feedback = db["feedback"]
    print("SUCCESS: Successfully connected to MongoDB!")
except Exception as e:
    print(f"FATAL: Error connecting to MongoDB: {e}. Exiting.")
    sys.exit(1)

@app.context_processor
def inject_ads():
    ad_codes = settings.find_one()
    return dict(ad_settings=(ad_codes or {}), bot_username=BOT_USERNAME, main_channel_link=MAIN_CHANNEL_LINK)

def delete_message_after_delay(chat_id, message_id):
    print(f"Attempting to delete message {message_id} from chat {chat_id}")
    try:
        url = f"{TELEGRAM_API_URL}/deleteMessage"
        payload = {'chat_id': chat_id, 'message_id': message_id}
        requests.post(url, json=payload)
    except Exception as e:
        print(f"Error in delete_message_after_delay: {e}")

scheduler = BackgroundScheduler(daemon=True)
scheduler.start()

def escape_markdown(text: str) -> str:
    if not isinstance(text, str):
        return ''
    escape_chars = r'_*[]()~`>#+-=|{}.!'
    return re.sub(f'([{re.escape(escape_chars)}])', r'\\\1', text)

# ======================================================================
# --- HTML টেমপ্লেট ---
# ======================================================================

index_html = """
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no" />
<title>MovieZone - Your Entertainment Hub</title>
<style>
  @import url('https://fonts.googleapis.com/css2?family=Bebas+Neue&family=Roboto:wght@400;500;700&display=swap');
  :root { --netflix-red: #E50914; --netflix-black: #141414; --text-light: #f5f5f5; --text-dark: #a0a0a0; --nav-height: 60px; }
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body { font-family: 'Roboto', sans-serif; background-color: var(--netflix-black); color: var(--text-light); overflow-x: hidden; }
  a { text-decoration: none; color: inherit; }
  ::-webkit-scrollbar { width: 8px; } ::-webkit-scrollbar-track { background: #222; } ::-webkit-scrollbar-thumb { background: #555; } ::-webkit-scrollbar-thumb:hover { background: var(--netflix-red); }
  .main-nav { position: fixed; top: 0; left: 0; width: 100%; padding: 15px 50px; display: flex; justify-content: space-between; align-items: center; z-index: 100; transition: background-color 0.3s ease; background: linear-gradient(to bottom, rgba(0,0,0,0.8) 10%, rgba(0,0,0,0)); }
  .main-nav.scrolled { background-color: var(--netflix-black); }
  .logo { font-family: 'Bebas Neue', sans-serif; font-size: 32px; color: var(--netflix-red); font-weight: 700; letter-spacing: 1px; }
  .search-input { background-color: rgba(0,0,0,0.7); border: 1px solid #777; color: var(--text-light); padding: 8px 15px; border-radius: 4px; transition: width 0.3s ease, background-color 0.3s ease; width: 250px; }
  .search-input:focus { background-color: rgba(0,0,0,0.9); border-color: var(--text-light); outline: none; }
  .tags-section { padding: 80px 50px 20px 50px; background-color: var(--netflix-black); }
  .tags-container { display: flex; flex-wrap: wrap; justify-content: center; gap: 10px; }
  .tag-link { padding: 6px 16px; background-color: rgba(255, 255, 255, 0.1); border: 1px solid #444; border-radius: 50px; font-weight: 500; font-size: 0.85rem; transition: all 0.3s; }
  .tag-link:hover { background-color: var(--netflix-red); border-color: var(--netflix-red); color: white; }
  .hero-section { height: 85vh; position: relative; color: white; overflow: hidden; }
  .hero-slide { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background-size: cover; background-position: center top; display: flex; align-items: flex-end; padding: 50px; opacity: 0; transition: opacity 1.5s ease-in-out; z-index: 1; }
  .hero-slide.active { opacity: 1; z-index: 2; }
  .hero-slide::before { content: ''; position: absolute; top: 0; left: 0; right: 0; bottom: 0; background: linear-gradient(to top, var(--netflix-black) 10%, transparent 50%), linear-gradient(to right, rgba(0,0,0,0.8) 0%, transparent 60%); }
  .hero-content { position: relative; z-index: 3; max-width: 50%; }
  .hero-title { font-family: 'Bebas Neue', sans-serif; font-size: 5rem; font-weight: 700; margin-bottom: 1rem; line-height: 1; }
  .hero-overview { font-size: 1.1rem; line-height: 1.5; margin-bottom: 1.5rem; max-width: 600px; display: -webkit-box; -webkit-line-clamp: 3; -webkit-box-orient: vertical; overflow: hidden; }
  .hero-buttons .btn { padding: 8px 20px; margin-right: 0.8rem; border: none; border-radius: 4px; font-size: 0.9rem; font-weight: 700; cursor: pointer; transition: opacity 0.3s ease; display: inline-flex; align-items: center; gap: 8px; }
  .btn.btn-primary { background-color: var(--netflix-red); color: white; } .btn.btn-secondary { background-color: rgba(109, 109, 110, 0.7); color: white; } .btn:hover { opacity: 0.8; }
  main { padding: 0 50px; }
  .movie-card { display: block; cursor: pointer; transition: transform 0.3s ease; }
  .poster-wrapper { position: relative; width: 100%; border-radius: 6px; overflow: hidden; background-color: #222; display: flex; flex-direction: column; }
  .movie-poster-container { position: relative; overflow: hidden; width:100%; flex-grow:1; aspect-ratio: 2 / 3; }
  .movie-poster { width: 100%; height: 100%; object-fit: cover; display: block; transition: transform 0.4s ease; }
  @keyframes rgb-glow {
    0%, 100% { color: #ff5555; text-shadow: 0 0 5px #ff5555, 0 0 10px #ff5555; }
    33% { color: #55ff55; text-shadow: 0 0 5px #55ff55, 0 0 10px #55ff55; }
    66% { color: #55aaff; text-shadow: 0 0 5px #55aaff, 0 0 10px #55aaff; }
  }
  .poster-badge {
    position: absolute; top: 18px; left: -35px; width: 140px; background: rgba(20, 20, 20, 0.8);
    backdrop-filter: blur(5px); transform: rotate(-45deg); text-align: center; z-index: 5;
    font-size: 0.75rem; font-weight: 700; padding: 4px 0; border: 1px solid rgba(255, 255, 255, 0.2);
    animation: rgb-glow 3s linear infinite;
  }
  .rating-badge { position: absolute; top: 10px; right: 10px; background-color: rgba(0, 0, 0, 0.8); color: white; padding: 5px 10px; font-size: 0.8rem; font-weight: 700; border-radius: 20px; z-index: 3; display: flex; align-items: center; gap: 5px; backdrop-filter: blur(5px); }
  .rating-badge .fa-star { color: #f5c518; }
  .card-info-static { padding: 10px 8px; background-color: #1a1a1a; text-align: left; width: 100%; flex-shrink: 0; }
  .card-info-title { font-size: 0.9rem; font-weight: 500; color: var(--text-light); margin: 0 0 4px 0; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  .card-info-meta { font-size: 0.75rem; color: var(--text-dark); margin: 0; }
  @media (hover: hover) { .movie-card:hover { transform: scale(1.05); z-index: 10; box-shadow: 0 0 20px rgba(229, 9, 20, 0.5); } .movie-card:hover .movie-poster { transform: scale(1.1); } }
  .full-page-grid-container { padding-top: 100px; padding-bottom: 50px; }
  .full-page-grid-title { font-size: 2.5rem; font-weight: 700; margin-bottom: 30px; }
  .category-grid, .full-page-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 20px 15px; }
  .category-section { margin: 40px 0; }
  .category-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px; }
  .category-title { font-family: 'Roboto', sans-serif; font-weight: 700; font-size: 1.6rem; margin: 0; }
  .see-all-link { color: var(--text-dark); font-weight: 700; font-size: 0.9rem; }
  .bottom-nav { display: none; position: fixed; bottom: 0; left: 0; right: 0; height: var(--nav-height); background-color: #181818; border-top: 1px solid #282828; justify-content: space-around; align-items: center; z-index: 200; }
  .nav-item { display: flex; flex-direction: column; align-items: center; color: var(--text-dark); font-size: 10px; flex-grow: 1; padding: 5px 0; transition: color 0.2s ease; }
  .nav-item i { font-size: 20px; margin-bottom: 4px; } .nav-item.active { color: var(--text-light); } .nav-item.active i { color: var(--netflix-red); }
  .ad-container { margin: 40px 0; display: flex; justify-content: center; align-items: center; }
  .telegram-join-section { background-color: #181818; padding: 40px 20px; text-align: center; margin: 50px -50px -50px -50px; }
  .telegram-join-section .telegram-icon { font-size: 4rem; color: #2AABEE; margin-bottom: 15px; } .telegram-join-section h2 { font-family: 'Bebas Neue', sans-serif; font-size: 2.5rem; color: var(--text-light); margin-bottom: 10px; }
  .telegram-join-section p { font-size: 1.1rem; color: var(--text-dark); max-width: 600px; margin: 0 auto 25px auto; }
  .telegram-join-button { display: inline-flex; align-items: center; gap: 10px; background-color: #2AABEE; color: white; padding: 12px 30px; border-radius: 50px; font-size: 1.1rem; font-weight: 700; transition: all 0.2s ease; }
  .telegram-join-button:hover { transform: scale(1.05); background-color: #1e96d1; } .telegram-join-button i { font-size: 1.3rem; }
  @media (max-width: 768px) {
      body { padding-bottom: var(--nav-height); } .main-nav { padding: 10px 15px; } main { padding: 0 15px; } .logo { font-size: 24px; } .search-input { width: 150px; }
      .tags-section { padding: 80px 15px 15px 15px; } .tag-link { padding: 6px 15px; font-size: 0.8rem; } .hero-section { height: 60vh; margin: 0 -15px;}
      .hero-slide { padding: 15px; align-items: center; } .hero-content { max-width: 90%; text-align: center; } .hero-title { font-size: 2.8rem; } .hero-overview { display: none; }
      .category-section { margin: 25px 0; } .category-title { font-size: 1.2rem; }
      .category-grid, .full-page-grid { grid-template-columns: repeat(auto-fill, minmax(110px, 1fr)); gap: 15px 10px; }
      .full-page-grid-container { padding-top: 80px; } .full-page-grid-title { font-size: 1.8rem; }
      .bottom-nav { display: flex; } .ad-container { margin: 25px 0; }
      .telegram-join-section { margin: 50px -15px -30px -15px; }
      .telegram-join-section h2 { font-size: 2rem; } .telegram-join-section p { font-size: 1rem; }
  }
</style>
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.2.0/css/all.min.css">
</head>
<body>
<header class="main-nav"><a href="{{ url_for('home') }}" class="logo">MovieZone</a><form method="GET" action="/" class="search-form"><input type="search" name="q" class="search-input" placeholder="Search..." value="{{ query|default('') }}" /></form></header>
<main>
  {% macro render_movie_card(m) %}
    <a href="{{ url_for('movie_detail', movie_id=m._id) }}" class="movie-card">
      <div class="poster-wrapper">
        <div class="movie-poster-container">
           <img class="movie-poster" loading="lazy" src="{{ m.poster or 'https://via.placeholder.com/400x600.png?text=No+Image' }}" alt="{{ m.title }}">
           {% if m.poster_badge %}<div class="poster-badge">{{ m.poster_badge }}</div>{% endif %}
           {% if m.vote_average and m.vote_average > 0 %}<div class="rating-badge"><i class="fas fa-star"></i> {{ "%.1f"|format(m.vote_average) }}</div>{% endif %}
        </div>
        <div class="card-info-static">
          <h4 class="card-info-title">{{ m.title }}</h4>
          {% if m.release_date %}<p class="card-info-meta">{{ m.release_date.split('-')[0] }}</p>{% endif %}
        </div>
      </div>
    </a>
  {% endmacro %}

  {% if is_full_page_list %}
    <div class="full-page-grid-container">
        <h2 class="full-page-grid-title">{{ query }}</h2>
        {% if movies|length == 0 %}
            <p style="text-align:center; color: var(--text-dark); margin-top: 40px;">No content found.</p>
        {% else %}
            <div class="full-page-grid">
                {% for m in movies %}
                    {{ render_movie_card(m) }}
                {% endfor %}
            </div>
        {% endif %}
    </div>
  {% else %}
    {% if all_badges %}<div class="tags-section"><div class="tags-container">{% for badge in all_badges %}<a href="{{ url_for('movies_by_badge', badge_name=badge) }}" class="tag-link">{{ badge }}</a>{% endfor %}</div></div>{% endif %}
    
    {% if recently_added %}<div class="hero-section">{% for movie in recently_added %}<div class="hero-slide {% if loop.first %}active{% endif %}" style="background-image: url('{{ movie.poster or '' }}');"><div class="hero-content"><h1 class="hero-title">{{ movie.title }}</h1><p class="hero-overview">{{ movie.overview }}</p><div class="hero-buttons">{% if movie.watch_link and not movie.is_coming_soon %}<a href="{{ url_for('watch_movie', movie_id=movie._id) }}" class="btn btn-primary"><i class="fas fa-play"></i> Watch Now</a>{% endif %}<a href="{{ url_for('movie_detail', movie_id=movie._id) }}" class="btn btn-secondary"><i class="fas fa-info-circle"></i> More Info</a></div></div></div>{% endfor %}</div>{% endif %}

    {% macro render_grid_section(title, movies_list, endpoint) %}
        {% if movies_list %}
        <div class="category-section">
            <div class="category-header">
                <h2 class="category-title">{{ title }}</h2>
                <a href="{{ url_for(endpoint) }}" class="see-all-link">See All ></a>
            </div>
            <div class="category-grid">
                {% for m in movies_list %}
                    {{ render_movie_card(m) }}
                {% endfor %}
            </div>
        </div>
        {% endif %}
    {% endmacro %}

    {{ render_grid_section('Trending Now', trending_movies, 'trending_movies') }}
    {% if ad_settings.banner_ad_code %}<div class="ad-container">{{ ad_settings.banner_ad_code|safe }}</div>{% endif %}
    {{ render_grid_section('Latest Movies', latest_movies, 'movies_only') }}
    {% if ad_settings.native_banner_code %}<div class="ad-container">{{ ad_settings.native_banner_code|safe }}</div>{% endif %}
    {{ render_grid_section('Web Series', latest_series, 'webseries') }}
    {{ render_grid_section('Recently Added', recently_added_full, 'recently_added_all') }}
    {{ render_grid_section('Coming Soon', coming_soon_movies, 'coming_soon') }}
    
    <div class="telegram-join-section">
        <i class="fa-brands fa-telegram telegram-icon"></i>
        <h2>Join Our Telegram Channel</h2>
        <p>Get the latest movie updates, news, and direct download links right on your phone!</p>
        <a href="{{ main_channel_link or '#' }}" target="_blank" class="telegram-join-button"><i class="fa-brands fa-telegram"></i> Join Main Channel</a>
    </div>
  {% endif %}
</main>
<nav class="bottom-nav">
    <a href="{{ url_for('home') }}" class="nav-item {% if request.endpoint == 'home' %}active{% endif %}">
        <i class="fas fa-home"></i><span>Home</span>
    </a>
    <a href="{{ url_for('movies_only') }}" class="nav-item {% if request.endpoint == 'movies_only' %}active{% endif %}">
        <i class="fas fa-film"></i><span>Movies</span>
    </a>
    <a href="{{ url_for('webseries') }}" class="nav-item {% if request.endpoint == 'webseries' %}active{% endif %}">
        <i class="fas fa-tv"></i><span>Series</span>
    </a>
    <a href="{{ url_for('genres_page') }}" class="nav-item {% if request.endpoint == 'genres_page' %}active{% endif %}">
        <i class="fas fa-layer-group"></i><span>Genres</span>
    </a>
    <a href="{{ url_for('contact') }}" class="nav-item {% if request.endpoint == 'contact' %}active{% endif %}">
        <i class="fas fa-envelope"></i><span>Request</span>
    </a>
</nav>
<script>
    const nav = document.querySelector('.main-nav');
    window.addEventListener('scroll', () => { window.scrollY > 50 ? nav.classList.add('scrolled') : nav.classList.remove('scrolled'); });
    document.addEventListener('DOMContentLoaded', function() { const slides = document.querySelectorAll('.hero-slide'); if (slides.length > 1) { let currentSlide = 0; const showSlide = (index) => slides.forEach((s, i) => s.classList.toggle('active', i === index)); setInterval(() => { currentSlide = (currentSlide + 1) % slides.length; showSlide(currentSlide); }, 5000); } });
</script>
{% if ad_settings.popunder_code %}{{ ad_settings.popunder_code|safe }}{% endif %}
{% if ad_settings.social_bar_code %}{{ ad_settings.social_bar_code|safe }}{% endif %}
</body>
</html>
"""

detail_html = """
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no" />
<title>{{ movie.title if movie else "Content Not Found" }} - MovieZone</title>
<style>
  @import url('https://fonts.googleapis.com/css2?family=Bebas+Neue&family=Roboto:wght@400;500;700&display=swap');
  :root { --netflix-red: #E50914; --netflix-black: #141414; --text-light: #f5f5f5; --text-dark: #a0a0a0; }
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body { font-family: 'Roboto', sans-serif; background: var(--netflix-black); color: var(--text-light); }
  .detail-header { position: absolute; top: 0; left: 0; right: 0; padding: 20px 50px; z-index: 100; }
  .back-button { color: var(--text-light); font-size: 1.2rem; font-weight: 700; text-decoration: none; display: flex; align-items: center; gap: 10px; transition: color 0.3s ease; }
  .back-button:hover { color: var(--netflix-red); }
  .detail-hero { position: relative; width: 100%; display: flex; align-items: center; justify-content: center; padding: 100px 0; }
  .detail-hero-background { position: absolute; top: 0; left: 0; right: 0; bottom: 0; background-size: cover; background-position: center; filter: blur(20px) brightness(0.4); transform: scale(1.1); }
  .detail-hero::after { content: ''; position: absolute; top: 0; left: 0; right: 0; bottom: 0; background: linear-gradient(to top, rgba(20,20,20,1) 0%, rgba(20,20,20,0.6) 50%, rgba(20,20,20,1) 100%); }
  .detail-content-wrapper { position: relative; z-index: 2; display: flex; gap: 40px; max-width: 1200px; padding: 0 50px; width: 100%; }
  .detail-poster { width: 300px; height: 450px; flex-shrink: 0; border-radius: 8px; box-shadow: 0 10px 30px rgba(0,0,0,0.5); object-fit: cover; }
  .detail-info { flex-grow: 1; max-width: 65%; }
  .detail-title { font-family: 'Bebas Neue', sans-serif; font-size: 4.5rem; font-weight: 700; line-height: 1.1; margin-bottom: 20px; }
  .detail-meta { display: flex; flex-wrap: wrap; gap: 20px; margin-bottom: 25px; font-size: 1rem; color: var(--text-dark); }
  .detail-meta span { font-weight: 700; color: var(--text-light); }
  .detail-meta span i { margin-right: 5px; color: var(--text-dark); }
  .detail-overview { font-size: 1.1rem; line-height: 1.6; margin-bottom: 30px; }
  .action-btn { background-color: var(--netflix-red); color: white; padding: 15px 30px; font-size: 1.2rem; font-weight: 700; border: none; border-radius: 5px; cursor: pointer; display: inline-flex; align-items: center; gap: 10px; text-decoration: none; margin-bottom: 15px; transition: all 0.2s ease; }
  .action-btn:hover { transform: scale(1.05); background-color: #f61f29; }
  .section-title { font-size: 1.5rem; font-weight: 700; margin-bottom: 20px; padding-bottom: 5px; border-bottom: 2px solid var(--netflix-red); display: inline-block; }
  .video-container { position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%; background: #000; border-radius: 8px; }
  .video-container iframe { position: absolute; top: 0; left: 0; width: 100%; height: 100%; }
  .download-section, .episode-section { margin-top: 30px; }
  .download-button, .episode-button { display: inline-block; padding: 12px 25px; background-color: #444; color: white; text-decoration: none; border-radius: 4px; font-weight: 700; transition: background-color 0.3s ease; margin-right: 10px; margin-bottom: 10px; text-align: center; vertical-align: middle; }
  .copy-button { background-color: #555; color: white; border: none; padding: 8px 15px; font-size: 0.9rem; cursor: pointer; border-radius: 4px; margin-left: -5px; margin-bottom: 10px; vertical-align: middle; }
  .episode-item { display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px; padding: 15px; border-radius: 5px; background-color: #1a1a1a; border-left: 4px solid var(--netflix-red); }
  .episode-title { font-size: 1.1rem; font-weight: 500; color: #fff; }
  .ad-container { margin: 30px 0; text-align: center; }
  .related-section-container { padding: 40px 0; background-color: #181818; }
  .related-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 20px 15px; padding: 0 50px; }
  .movie-card { display: block; cursor: pointer; transition: transform 0.3s ease; }
  .poster-wrapper { position: relative; width: 100%; border-radius: 6px; overflow: hidden; background-color: #222; display: flex; flex-direction: column; }
  .movie-poster-container { position: relative; overflow: hidden; width:100%; flex-grow:1; aspect-ratio: 2 / 3; }
  .movie-poster { width: 100%; height: 100%; object-fit: cover; display: block; transition: transform 0.4s ease; }
  @keyframes rgb-glow {
    0%, 100% { color: #ff5555; text-shadow: 0 0 5px #ff5555, 0 0 10px #ff5555; }
    33% { color: #55ff55; text-shadow: 0 0 5px #55ff55, 0 0 10px #55ff55; }
    66% { color: #55aaff; text-shadow: 0 0 5px #55aaff, 0 0 10px #55aaff; }
  }
  .poster-badge {
    position: absolute; top: 18px; left: -35px; width: 140px; background: rgba(20, 20, 20, 0.8);
    backdrop-filter: blur(5px); transform: rotate(-45deg); text-align: center; z-index: 5;
    font-size: 0.75rem; font-weight: 700; padding: 4px 0; border: 1px solid rgba(255, 255, 255, 0.2);
    animation: rgb-glow 3s linear infinite;
  }
  .rating-badge { position: absolute; top: 10px; right: 10px; background-color: rgba(0, 0, 0, 0.8); color: white; padding: 5px 10px; font-size: 0.8rem; font-weight: 700; border-radius: 20px; z-index: 3; display: flex; align-items: center; gap: 5px; backdrop-filter: blur(5px); }
  .rating-badge .fa-star { color: #f5c518; }
  .card-info-static { padding: 10px 8px; background-color: #1a1a1a; text-align: left; width: 100%; flex-shrink: 0; }
  .card-info-title { font-size: 0.9rem; font-weight: 500; color: var(--text-light); margin: 0 0 4px 0; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  .card-info-meta { font-size: 0.75rem; color: var(--text-dark); margin: 0; }
  @media (hover: hover) { .movie-card:hover { transform: scale(1.05); z-index: 10; box-shadow: 0 0 20px rgba(229, 9, 20, 0.5); } .movie-card:hover .movie-poster { transform: scale(1.1); } }
  @media (max-width: 992px) { .detail-content-wrapper { flex-direction: column; align-items: center; text-align: center; } .detail-info { max-width: 100%; } .detail-title { font-size: 3.5rem; } }
  @media (max-width: 768px) { .detail-header { padding: 20px; } .detail-hero { padding: 80px 20px 40px; } .detail-poster { width: 60%; max-width: 220px; height: auto; } .detail-title { font-size: 2.2rem; }
  .action-btn, .download-button { display: block; width: 100%; max-width: 320px; margin: 0 auto 10px auto; }
  .episode-item { flex-direction: column; align-items: flex-start; gap: 10px; } .episode-button { width: 100%; }
  .section-title { margin-left: 15px !important; } .related-section-container { padding: 20px 0; }
  .related-grid { grid-template-columns: repeat(auto-fill, minmax(110px, 1fr)); gap: 15px 10px; padding: 0 15px; } }
</style>
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.2.0/css/all.min.css">
</head>
<body>
{% macro render_movie_card(m) %}
  <a href="{{ url_for('movie_detail', movie_id=m._id) }}" class="movie-card">
    <div class="poster-wrapper">
      <div class="movie-poster-container">
        <img class="movie-poster" loading="lazy" src="{{ m.poster or 'https://via.placeholder.com/400x600.png?text=No+Image' }}" alt="{{ m.title }}">
        {% if m.poster_badge %}<div class="poster-badge">{{ m.poster_badge }}</div>{% endif %}
        {% if m.vote_average and m.vote_average > 0 %}<div class="rating-badge"><i class="fas fa-star"></i> {{ "%.1f"|format(m.vote_average) }}</div>{% endif %}
      </div>
      <div class="card-info-static">
        <h4 class="card-info-title">{{ m.title }}</h4>
        {% if m.release_date %}<p class="card-info-meta">{{ m.release_date.split('-')[0] }}</p>{% endif %}
      </div>
    </div>
  </a>
{% endmacro %}
<header class="detail-header"><a href="{{ url_for('home') }}" class="back-button"><i class="fas fa-arrow-left"></i> Back to Home</a></header>
{% if movie %}
<div class="detail-hero" style="min-height: auto; padding-bottom: 60px;">
  <div class="detail-hero-background" style="background-image: url('{{ movie.poster }}');"></div>
  <div class="detail-content-wrapper"><img class="detail-poster" src="{{ movie.poster or 'https://via.placeholder.com/400x600.png?text=No+Image' }}" alt="{{ movie.title }}">
    <div class="detail-info">
      <h1 class="detail-title">{{ movie.title }}</h1>
      <div class="detail-meta">
        {% if movie.release_date %}<span>{{ movie.release_date.split('-')[0] }}</span>{% endif %}
        {% if movie.vote_average %}<span><i class="fas fa-star" style="color:#f5c518;"></i> {{ "%.1f"|format(movie.vote_average) }}</span>{% endif %}
        {% if movie.languages %}<span><i class="fas fa-language"></i> {{ movie.languages | join(' • ') }}</span>{% endif %}
        {% if movie.genres %}<span>{{ movie.genres | join(' • ') }}</span>{% endif %}
      </div>
      <p class="detail-overview">{{ movie.overview }}</p>
      {% if movie.type == 'movie' and movie.watch_link %}<a href="{{ url_for('watch_movie', movie_id=movie._id) }}" class="action-btn"><i class="fas fa-play"></i> Watch Now</a>{% endif %}
      {% if ad_settings.banner_ad_code %}<div class="ad-container">{{ ad_settings.banner_ad_code|safe }}</div>{% endif %}
      {% if trailer_key %}<div class="trailer-section"><h3 class="section-title">Watch Trailer</h3><div class="video-container"><iframe src="https://www.youtube.com/embed/{{ trailer_key }}" frameborder="0" allowfullscreen></iframe></div></div>{% endif %}
      <div style="margin: 20px 0;"><a href="{{ url_for('contact', report_id=movie._id, title=movie.title) }}" class="download-button" style="background-color:#5a5a5a; text-align:center;"><i class="fas fa-flag"></i> Report a Problem</a></div>
      {% if movie.is_coming_soon %}<h3 class="section-title">Coming Soon</h3>
      {% elif movie.type == 'movie' %}
        <div class="download-section">
          {% if movie.links %}<h3 class="section-title">Download Links</h3>{% for link_item in movie.links %}<div><a class="download-button" href="{{ link_item.url }}" target="_blank" rel="noopener"><i class="fas fa-download"></i> {{ link_item.quality }}</a><button class="copy-button" onclick="copyToClipboard('{{ link_item.url }}')"><i class="fas fa-copy"></i></button></div>{% endfor %}{% endif %}
          {% if movie.files %}<h3 class="section-title">Get from Telegram</h3>{% for file in movie.files | sort(attribute='quality') %}<a href="https://t.me/{{ bot_username }}?start={{ movie._id }}_{{ file.quality }}" class="action-btn" style="background-color: #2AABEE; display: block; text-align:center; margin-top:10px; margin-bottom: 0;"><i class="fa-brands fa-telegram"></i> Get {{ file.quality }}</a>{% endfor %}{% endif %}
        </div>
      {% elif movie.type == 'series' %}
        <div class="episode-section">
          <h3 class="section-title">Episodes & Seasons</h3>
          
          {# --- Season Packs --- #}
          {% if movie.season_packs %}
            {% for pack in movie.season_packs | sort(attribute='season') %}
              <div class="episode-item" style="background-color: #3e1a1a;">
                <span class="episode-title">Complete Season {{ pack.season }} Pack</span>
                <a href="https://t.me/{{ bot_username }}?start={{ movie._id }}_S{{ pack.season }}" class="episode-button" style="background-color: var(--netflix-red);"><i class="fas fa-box-open"></i> Get Season Pack</a>
              </div>
            {% endfor %}
          {% endif %}

          {# --- Individual Episodes --- #}
          {% if movie.episodes %}
            {% for ep in movie.episodes | sort(attribute='episode_number') | sort(attribute='season') %}
              <div class="episode-item">
                <span class="episode-title">Season {{ ep.season }} - Episode {{ ep.episode_number }}</span>
                <a href="https://t.me/{{ bot_username }}?start={{ movie._id }}_{{ ep.season }}_{{ ep.episode_number }}" class="episode-button" style="background-color: #2AABEE;"><i class="fa-brands fa-telegram"></i> Get Episode</a>
              </div>
            {% endfor %}
          {% endif %}

          {% if not movie.episodes and not movie.season_packs %}
             <p>No episodes or season packs available yet.</p>
          {% endif %}
        </div>
      {% endif %}
    </div>
  </div>
</div>
{% if related_movies %}<div class="related-section-container"><h3 class="section-title" style="margin-left: 50px; color: white;">You Might Also Like</h3><div class="related-grid">{% for m in related_movies %}{{ render_movie_card(m) }}{% endfor %}</div></div>{% endif %}
{% else %}<div style="display:flex; justify-content:center; align-items:center; height:100vh;"><h2>Content not found.</h2></div>{% endif %}
<script>
function copyToClipboard(text) { navigator.clipboard.writeText(text).then(() => alert('Link copied!'), () => alert('Copy failed!')); }
</script>
{% if ad_settings.popunder_code %}{{ ad_settings.popunder_code|safe }}{% endif %}
{% if ad_settings.social_bar_code %}{{ ad_settings.social_bar_code|safe }}{% endif %}
</body>
</html>
"""

genres_html = """
<!DOCTYPE html>
<html lang="en"><head><meta charset="UTF-8" /><meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no" /><title>{{ title }} - MovieZone</title>
<style>
  @import url('https://fonts.googleapis.com/css2?family=Bebas+Neue&family=Roboto:wght@400;500;700&display=swap');
  :root { --netflix-red: #E50914; --netflix-black: #141414; --text-light: #f5f5f5; }
  * { box-sizing: border-box; margin: 0; padding: 0; } body { font-family: 'Roboto', sans-serif; background-color: var(--netflix-black); color: var(--text-light); } a { text-decoration: none; color: inherit; }
  .main-container { padding: 100px 50px 50px; } .page-title { font-family: 'Bebas Neue', sans-serif; font-size: 3rem; color: var(--netflix-red); margin-bottom: 30px; }
  .back-button { color: var(--text-light); font-size: 1rem; margin-bottom: 20px; display: inline-block; } .back-button:hover { color: var(--netflix-red); }
  .genre-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 20px; }
  .genre-card { background: linear-gradient(45deg, #2c2c2c, #1a1a1a); border-radius: 8px; padding: 30px 20px; text-align: center; font-size: 1.4rem; font-weight: 700; transition: all 0.3s ease; border: 1px solid #444; }
  .genre-card:hover { transform: translateY(-5px) scale(1.03); background: linear-gradient(45deg, var(--netflix-red), #b00710); border-color: var(--netflix-red); }
  @media (max-width: 768px) { .main-container { padding: 80px 15px 30px; } .page-title { font-size: 2.2rem; } .genre-grid { grid-template-columns: repeat(2, 1fr); gap: 15px; } .genre-card { font-size: 1.1rem; padding: 25px 15px; } }
</style><link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.2.0/css/all.min.css"></head>
<body>
<div class="main-container"><a href="{{ url_for('home') }}" class="back-button"><i class="fas fa-arrow-left"></i> Back to Home</a><h1 class="page-title">{{ title }}</h1>
<div class="genre-grid">{% for genre in genres %}<a href="{{ url_for('movies_by_genre', genre_name=genre) }}" class="genre-card"><span>{{ genre }}</span></a>{% endfor %}</div></div>
{% if ad_settings.popunder_code %}{{ ad_settings.popunder_code|safe }}{% endif %}
{% if ad_settings.social_bar_code %}{{ ad_settings.social_bar_code|safe }}{% endif %}
</body></html>
"""

watch_html = """
<!DOCTYPE html>
<html lang="en"><head><meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1.0"><title>Watching: {{ title }}</title>
<style> body, html { margin: 0; padding: 0; height: 100%; overflow: hidden; background-color: #000; } .player-container { width: 100%; height: 100%; } .player-container iframe { width: 100%; height: 100%; border: 0; } </style></head>
<body><div class="player-container"><iframe src="{{ watch_link }}" allowfullscreen allowtransparency allow="autoplay" scrolling="no" frameborder="0"></iframe></div>
{% if ad_settings.popunder_code %}{{ ad_settings.popunder_code|safe }}{% endif %}
{% if ad_settings.social_bar_code %}{{ ad_settings.social_bar_code|safe }}{% endif %}
</body></html>
"""

admin_html = """
<!DOCTYPE html>
<html><head><title>Admin Panel - MovieZone</title><meta name="viewport" content="width=device-width, initial-scale=1" /><style>
:root { --netflix-red: #E50914; --netflix-black: #141414; --dark-gray: #222; --light-gray: #333; --text-light: #f5f5f5; }
body { font-family: 'Roboto', sans-serif; background: var(--netflix-black); color: var(--text-light); padding: 20px; }
h2, h3 { font-family: 'Bebas Neue', sans-serif; color: var(--netflix-red); } h2 { font-size: 2.5rem; margin-bottom: 20px; } h3 { font-size: 1.5rem; margin: 20px 0 10px 0;}
form { max-width: 800px; margin: 0 auto 40px auto; background: var(--dark-gray); padding: 25px; border-radius: 8px;}
.form-group { margin-bottom: 15px; } .form-group label { display: block; margin-bottom: 8px; font-weight: bold; }
input[type="text"], input[type="url"], input[type="search"], textarea, select, input[type="number"], input[type="email"] { width: 100%; padding: 12px; border-radius: 4px; border: 1px solid var(--light-gray); font-size: 1rem; background: var(--light-gray); color: var(--text-light); box-sizing: border-box; }
input[type="checkbox"] { width: auto; margin-right: 10px; transform: scale(1.2); } textarea { resize: vertical; min-height: 100px; }
button[type="submit"], .add-btn, .clear-btn { background: var(--netflix-red); color: white; font-weight: 700; cursor: pointer; border: none; padding: 12px 25px; border-radius: 4px; font-size: 1rem; transition: background 0.3s ease; text-decoration: none; }
button[type="submit"]:hover, .add-btn:hover { background: #b00710; }
.clear-btn { background: #555; display: inline-block; } .clear-btn:hover { background: #444; }
table { display: block; overflow-x: auto; white-space: nowrap; width: 100%; border-collapse: collapse; margin-top: 20px; }
th, td { padding: 12px 15px; text-align: left; border-bottom: 1px solid var(--light-gray); } th { background: #252525; } td { background: var(--dark-gray); }
.action-buttons { display: flex; gap: 10px; } .action-buttons a, .action-buttons button, .delete-btn { padding: 6px 12px; border-radius: 4px; text-decoration: none; color: white; border: none; cursor: pointer; }
.edit-btn { background: #007bff; } .delete-btn { background: #dc3545; }
.dynamic-item { border: 1px solid var(--light-gray); padding: 15px; margin-bottom: 15px; border-radius: 5px; }
hr.section-divider { border: 0; height: 2px; background-color: var(--light-gray); margin: 40px 0; }
.danger-zone { border: 2px solid var(--netflix-red); padding: 20px; border-radius: 8px; margin-top: 20px; text-align: center; }
.danger-zone-btn { background: #dc3545; color: white; text-decoration: none; padding: 10px 20px; border-radius: 5px; font-weight: bold; }
</style><link href="https://fonts.googleapis.com/css2?family=Bebas+Neue&family=Roboto:wght@400;700&display=swap" rel="stylesheet"></head>
<body>
  <h2>বিজ্ঞাপন পরিচালনা (Ad Management)</h2>
  <form action="{{ url_for('save_ads') }}" method="post"><div class="form-group"><label>Pop-Under / OnClick Ad Code</label><textarea name="popunder_code" rows="4">{{ ad_settings.popunder_code or '' }}</textarea></div><div class="form-group"><label>Social Bar / Sticky Ad Code</label><textarea name="social_bar_code" rows="4">{{ ad_settings.social_bar_code or '' }}</textarea></div><div class="form-group"><label>ব্যানার বিজ্ঞাপন কোড (Banner Ad)</label><textarea name="banner_ad_code" rows="4">{{ ad_settings.banner_ad_code or '' }}</textarea></div><div class="form-group"><label>নেটিভ ব্যানার বিজ্ঞাপন (Native Banner)</label><textarea name="native_banner_code" rows="4">{{ ad_settings.native_banner_code or '' }}</textarea></div><button type="submit">Save Ad Codes</button></form>
  <hr class="section-divider">
  <h2>Add New Content (Manual)</h2>
  <form method="post" action="{{ url_for('admin') }}">
    <div class="form-group"><label>Title (Required):</label><input type="text" name="title" required /></div>
    <div class="form-group"><label>Content Type:</label><select name="content_type" id="content_type" onchange="toggleFields()"><option value="movie">Movie</option><option value="series">TV/Web Series</option></select></div>
    <div id="movie_fields">
      <div class="form-group"><label>Watch Link (Embed URL):</label><input type="url" name="watch_link" /></div><hr><p><b>OR</b> Download Links (Manual)</p>
      <div class="form-group"><label>480p Link:</label><input type="url" name="link_480p" /></div>
      <div class="form-group"><label>720p Link:</label><input type="url" name="link_720p" /></div>
      <div class="form-group"><label>1080p Link:</label><input type="url" name="link_1080p" /></div>
      <hr><p><b>OR</b> Get from Telegram</p>
      <div id="telegram_files_container"></div><button type="button" onclick="addTelegramFileField()" class="add-btn">Add Telegram File</button>
    </div>
    <div id="episode_fields" style="display: none;">
      <h3>Episodes</h3><div id="episodes_container"></div>
      <button type="button" onclick="addEpisodeField()" class="add-btn">Add Episode</button>
    </div>
    <hr style="margin: 20px 0;"><button type="submit">Add Content</button>
  </form>
  <hr class="section-divider">
  <h2>Manage Content</h2>
  <form method="GET" action="{{ url_for('admin') }}" style="padding: 15px; background: #252525; display: flex; gap: 10px; align-items: center;">
    <input type="search" name="search" placeholder="Search by title..." value="{{ search_query or '' }}" style="flex-grow: 1;">
    <button type="submit">Search</button>
    {% if search_query %}<a href="{{ url_for('admin') }}" class="clear-btn">Clear</a>{% endif %}
  </form>
  <table><thead><tr><th>Title</th><th>Type</th><th>Actions</th></tr></thead><tbody>
    {% for movie in content_list %}
    <tr><td>{{ movie.title }}</td><td>{{ movie.type | title }}</td><td class="action-buttons"><a href="{{ url_for('edit_movie', movie_id=movie._id) }}" class="edit-btn">Edit</a><button class="delete-btn" onclick="confirmDelete('{{ movie._id }}', '{{ movie.title }}')">Delete</button></td></tr>
    {% else %}
    <tr><td colspan="3" style="text-align: center;">No content found.</td></tr>
    {% endfor %}
  </tbody></table>
  
  <div class="danger-zone">
      <h3>DANGER ZONE</h3>
      <p style="margin-bottom: 15px;">This will permanently delete all movies and series from the database. This action cannot be undone.</p>
      <a href="{{ url_for('delete_all_movies') }}" class="danger-zone-btn" onclick="return confirm('ARE YOU ABSOLUTELY SURE?\\nThis will delete ALL content from the database permanently.\\nThis action cannot be undone.');">Delete All Content</a>
  </div>

  <hr class="section-divider">
  <h2>User Feedback / Reports</h2>
  {% if feedback_list %}<table><thead><tr><th>Date</th><th>Type</th><th>Title</th><th>Message</th><th>Email</th><th>Action</th></tr></thead><tbody>{% for item in feedback_list %}<tr><td style="min-width: 150px;">{{ item.timestamp.strftime('%Y-%m-%d %H:%M') }}</td><td>{{ item.type }}</td><td>{{ item.content_title }}</td><td style="white-space: pre-wrap; min-width: 300px;">{{ item.message }}</td><td>{{ item.email or 'N/A' }}</td><td><a href="{{ url_for('delete_feedback', feedback_id=item._id) }}" class="delete-btn" onclick="return confirm('Delete this feedback?');">Delete</a></td></tr>{% endfor %}</tbody></table>{% else %}<p>No new feedback or reports.</p>{% endif %}
  <script>
    function confirmDelete(id, title) { if (confirm('Delete "' + title + '"?')) window.location.href = '/delete_movie/' + id; }
    function toggleFields() { var isSeries = document.getElementById('content_type').value === 'series'; document.getElementById('episode_fields').style.display = isSeries ? 'block' : 'none'; document.getElementById('movie_fields').style.display = isSeries ? 'none' : 'block'; }
    function addTelegramFileField() { const c = document.getElementById('telegram_files_container'); const d = document.createElement('div'); d.className = 'dynamic-item'; d.innerHTML = `<div class="form-group"><label>Quality (e.g., 720p):</label><input type="text" name="telegram_quality[]" required /></div><div class="form-group"><label>Message ID:</label><input type="number" name="telegram_message_id[]" required /></div><button type="button" onclick="this.parentElement.remove()" class="delete-btn">Remove</button>`; c.appendChild(d); }
    function addEpisodeField() { const c = document.getElementById('episodes_container'); const d = document.createElement('div'); d.className = 'dynamic-item'; d.innerHTML = `<div class="form-group"><label>Season Number:</label><input type="number" name="episode_season[]" value="1" required /></div><div class="form-group"><label>Episode Number:</label><input type="number" name="episode_number[]" required /></div><div class="form-group"><label>Episode Title:</label><input type="text" name="episode_title[]" /></div><hr><p><b>Provide ONE of the following:</b></p><div class="form-group"><label>Telegram Message ID:</label><input type="number" name="episode_message_id[]" /></div><p><b>OR</b> Watch Link:</p><div class="form-group"><label>Watch Link (Embed):</label><input type="url" name="episode_watch_link[]" /></div><button type="button" onclick="this.parentElement.remove()" class="delete-btn">Remove Episode</button>`; c.appendChild(d); }
    document.addEventListener('DOMContentLoaded', toggleFields);
  </script>
</body></html>
"""

edit_html = """
<!DOCTYPE html>
<html><head><title>Edit Content - MovieZone</title><meta name="viewport" content="width=device-width, initial-scale=1" /><style>
:root { --netflix-red: #E50914; --netflix-black: #141414; --dark-gray: #222; --light-gray: #333; --text-light: #f5f5f5; }
body { font-family: 'Roboto', sans-serif; background: var(--netflix-black); color: var(--text-light); padding: 20px; }
h2, h3 { font-family: 'Bebas Neue', sans-serif; color: var(--netflix-red); } h2 { font-size: 2.5rem; margin-bottom: 20px; } h3 { font-size: 1.5rem; margin: 20px 0 10px 0;}
form { max-width: 800px; margin: 0 auto 40px auto; background: var(--dark-gray); padding: 25px; border-radius: 8px;}
.form-group { margin-bottom: 15px; } .form-group label { display: block; margin-bottom: 8px; font-weight: bold; }
input, textarea, select { width: 100%; padding: 12px; border-radius: 4px; border: 1px solid var(--light-gray); font-size: 1rem; background: var(--light-gray); color: var(--text-light); box-sizing: border-box; }
input[type="checkbox"] { width: auto; margin-right: 10px; transform: scale(1.2); } textarea { resize: vertical; min-height: 100px; }
button[type="submit"], .add-btn { background: var(--netflix-red); color: white; font-weight: 700; cursor: pointer; border: none; padding: 12px 25px; border-radius: 4px; font-size: 1rem; }
.back-to-admin { display: inline-block; margin-bottom: 20px; color: var(--netflix-red); text-decoration: none; font-weight: bold; }
.dynamic-item { border: 1px solid var(--light-gray); padding: 15px; margin-bottom: 15px; border-radius: 5px; } .delete-btn { background: #dc3545; color: white; border: none; padding: 6px 12px; border-radius: 4px; cursor: pointer; }
</style><link href="https://fonts.googleapis.com/css2?family=Bebas+Neue&family=Roboto:wght@400;700&display=swap" rel="stylesheet"></head>
<body>
  <a href="{{ url_for('admin') }}" class="back-to-admin">← Back to Admin</a>
  <h2>Edit: {{ movie.title }}</h2>
  <form method="post">
    <div class="form-group"><label>Title:</label><input type="text" name="title" value="{{ movie.title }}" required /></div>
    <div class="form-group"><label>Poster URL:</label><input type="url" name="poster" value="{{ movie.poster or '' }}" /></div><div class="form-group"><label>Overview:</label><textarea name="overview">{{ movie.overview or '' }}</textarea></div>
    <div class="form-group"><label>Genres (comma separated):</label><input type="text" name="genres" value="{{ movie.genres|join(', ') if movie.genres else '' }}" /></div>
    <div class="form-group"><label>Languages (comma separated):</label><input type="text" name="languages" value="{{ movie.languages|join(', ') if movie.languages else '' }}" placeholder="e.g. Hindi, English, Bangla" /></div>
    <div class="form-group"><label>Poster Badge:</label><input type="text" name="poster_badge" value="{{ movie.poster_badge or '' }}" /></div>
    <div class="form-group"><label>Content Type:</label><select name="content_type" id="content_type" onchange="toggleFields()"><option value="movie" {% if movie.type == 'movie' %}selected{% endif %}>Movie</option><option value="series" {% if movie.type == 'series' %}selected{% endif %}>TV/Web Series</option></select></div>
    
    <div id="movie_fields">
        <div class="form-group"><label>Watch Link:</label><input type="url" name="watch_link" value="{{ movie.watch_link or '' }}" /></div><hr><p><b>OR</b> Download Links (Manual)</p>
        <div class="form-group"><label>480p Link:</label><input type="url" name="link_480p" value="{% for l in movie.links %}{% if l.quality == '480p' %}{{ l.url }}{% endif %}{% endfor %}" /></div>
        <div class="form-group"><label>720p Link:</label><input type="url" name="link_720p" value="{% for l in movie.links %}{% if l.quality == '720p' %}{{ l.url }}{% endif %}{% endfor %}" /></div>
        <div class="form-group"><label>1080p Link:</label><input type="url" name="link_1080p" value="{% for l in movie.links %}{% if l.quality == '1080p' %}{{ l.url }}{% endif %}{% endfor %}" /></div>
        <hr><p><b>OR</b> Get from Telegram</p>
        <div id="telegram_files_container">
            {% if movie.type == 'movie' and movie.files %}{% for file in movie.files %}
            <div class="dynamic-item">
                <div class="form-group"><label>Quality:</label><input type="text" name="telegram_quality[]" value="{{ file.quality }}" required /></div>
                <div class="form-group"><label>Message ID:</label><input type="number" name="telegram_message_id[]" value="{{ file.message_id }}" required /></div>
                <button type="button" onclick="this.parentElement.remove()" class="delete-btn">Remove</button>
            </div>
            {% endfor %}{% endif %}
        </div><button type="button" onclick="addTelegramFileField()" class="add-btn">Add Telegram File</button>
    </div>

    <div id="episode_fields" style="display: none;">
      <h3>Season Packs</h3>
      <div id="season_packs_container">
        {% if movie.type == 'series' and movie.season_packs %}
          {% for pack in movie.season_packs | sort(attribute='season') %}
          <div class="dynamic-item">
            <div class="form-group"><label>Season Number:</label><input type="number" name="pack_season[]" value="{{ pack.season }}" required /></div>
            <div class="form-group"><label>Telegram Message ID:</label><input type="number" name="pack_message_id[]" value="{{ pack.message_id }}" required /></div>
            <button type="button" onclick="this.parentElement.remove()" class="delete-btn">Remove Pack</button>
          </div>
          {% endfor %}
        {% endif %}
      </div>
      <button type="button" onclick="addSeasonPackField()" class="add-btn">Add Season Pack</button>
      <hr style="margin: 20px 0;">

      <h3>Individual Episodes</h3>
      <div id="episodes_container">
      {% if movie.type == 'series' and movie.episodes %}{% for ep in movie.episodes | sort(attribute='episode_number') | sort(attribute='season') %}<div class="dynamic-item">
        <div class="form-group"><label>Season Number:</label><input type="number" name="episode_season[]" value="{{ ep.season or 1 }}" required /></div>
        <div class="form-group"><label>Ep Number:</label><input type="number" name="episode_number[]" value="{{ ep.episode_number }}" required /></div>
        <div class="form-group"><label>Ep Title:</label><input type="text" name="episode_title[]" value="{{ ep.title or '' }}" /></div>
        <hr><p><b>Provide ONE of the following:</b></p>
        <div class="form-group"><label>Telegram Message ID:</label><input type="number" name="episode_message_id[]" value="{{ ep.message_id or '' }}" /></div>
        <p><b>OR</b> Watch Link:</p>
        <div class="form-group"><label>Watch Link (Embed):</label><input type="url" name="episode_watch_link[]" value="{{ ep.watch_link or '' }}" /></div>
        <button type="button" onclick="this.parentElement.remove()" class="delete-btn">Remove Episode</button>
      </div>{% endfor %}{% endif %}</div><button type="button" onclick="addEpisodeField()" class="add-btn">Add Episode</button>
    </div>
    
    <hr style="margin: 20px 0;">
    <div class="form-group"><input type="checkbox" name="is_trending" value="true" {% if movie.is_trending %}checked{% endif %}><label style="display: inline-block;">Is Trending?</label></div>
    <div class="form-group"><input type="checkbox" name="is_coming_soon" value="true" {% if movie.is_coming_soon %}checked{% endif %}><label style="display: inline-block;">Is Coming Soon?</label></div>
    <button type="submit">Update Content</button>
  </form>
  
  <script>
    function toggleFields() { var isSeries = document.getElementById('content_type').value === 'series'; document.getElementById('episode_fields').style.display = isSeries ? 'block' : 'none'; document.getElementById('movie_fields').style.display = isSeries ? 'none' : 'block'; }
    function addTelegramFileField() { const c = document.getElementById('telegram_files_container'); const d = document.createElement('div'); d.className = 'dynamic-item'; d.innerHTML = `<div class="form-group"><label>Quality (e.g., 720p):</label><input type="text" name="telegram_quality[]" required /></div><div class="form-group"><label>Message ID:</label><input type="number" name="telegram_message_id[]" required /></div><button type="button" onclick="this.parentElement.remove()" class="delete-btn">Remove</button>`; c.appendChild(d); }
    function addEpisodeField() { const c = document.getElementById('episodes_container'); const d = document.createElement('div'); d.className = 'dynamic-item'; d.innerHTML = `<div class="form-group"><label>Season Number:</label><input type="number" name="episode_season[]" value="1" required /></div><div class="form-group"><label>Episode Number:</label><input type="number" name="episode_number[]" required /></div><div class="form-group"><label>Episode Title:</label><input type="text" name="episode_title[]" /></div><hr><p><b>Provide ONE of the following:</b></p><div class="form-group"><label>Telegram Message ID:</label><input type="number" name="episode_message_id[]" /></div><p><b>OR</b> Watch Link:</p><div class="form-group"><label>Watch Link (Embed):</label><input type="url" name="episode_watch_link[]" /></div><button type="button" onclick="this.parentElement.remove()" class="delete-btn">Remove Episode</button>`; c.appendChild(d); }
    function addSeasonPackField() { const c = document.getElementById('season_packs_container'); const d = document.createElement('div'); d.className = 'dynamic-item'; d.innerHTML = `<div class="form-group"><label>Season Number:</label><input type="number" name="pack_season[]" required /></div><div class="form-group"><label>Telegram Message ID:</label><input type="number" name="pack_message_id[]" required /></div><button type="button" onclick="this.parentElement.remove()" class="delete-btn">Remove Pack</button>`; c.appendChild(d); }
    document.addEventListener('DOMContentLoaded', toggleFields);
  </script>
</body></html>
"""

contact_html = """
<!DOCTYPE html>
<html lang="bn"><head><meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1.0"><title>Contact Us / Report - MovieZone</title><style>
:root { --netflix-red: #E50914; --netflix-black: #141414; --dark-gray: #222; --light-gray: #333; --text-light: #f5f5f5; }
body { font-family: 'Roboto', sans-serif; background: var(--netflix-black); color: var(--text-light); padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }
.contact-container { max-width: 600px; width: 100%; background: var(--dark-gray); padding: 30px; border-radius: 8px; }
h2 { font-family: 'Bebas Neue', sans-serif; color: var(--netflix-red); font-size: 2.5rem; text-align: center; margin-bottom: 25px; }
.form-group { margin-bottom: 20px; } label { display: block; margin-bottom: 8px; font-weight: bold; }
input, select, textarea { width: 100%; padding: 12px; border-radius: 4px; border: 1px solid var(--light-gray); font-size: 1rem; background: var(--light-gray); color: var(--text-light); box-sizing: border-box; }
textarea { resize: vertical; min-height: 120px; } button[type="submit"] { background: var(--netflix-red); color: white; font-weight: 700; cursor: pointer; border: none; padding: 12px 25px; border-radius: 4px; font-size: 1.1rem; width: 100%; }
.success-message { text-align: center; padding: 20px; background-color: #1f4e2c; color: #d4edda; border-radius: 5px; margin-bottom: 20px; }
.back-link { display: block; text-align: center; margin-top: 20px; color: var(--netflix-red); text-decoration: none; font-weight: bold; }
</style><link href="https://fonts.googleapis.com/css2?family=Bebas+Neue&family=Roboto:wght@400;700&display=swap" rel="stylesheet"></head>
<body><div class="contact-container"><h2>Contact Us</h2>
{% if message_sent %}<div class="success-message"><p>আপনার বার্তা সফলভাবে পাঠানো হয়েছে। ধন্যবাদ!</p></div><a href="{{ url_for('home') }}" class="back-link">← Back to Home</a>
{% else %}<form method="post"><div class="form-group"><label for="type">বিষয় (Subject):</label><select name="type" id="type"><option value="Movie Request" {% if prefill_type == 'Problem Report' %}disabled{% endif %}>Movie/Series Request</option><option value="Problem Report" {% if prefill_type == 'Problem Report' %}selected{% endif %}>Report a Problem</option><option value="General Feedback">General Feedback</option></select></div><div class="form-group"><label for="content_title">মুভি/সিরিজের নাম (Title):</label><input type="text" name="content_title" id="content_title" value="{{ prefill_title }}" required></div><div class="form-group"><label for="message">আপনার বার্তা (Message):</label><textarea name="message" id="message" required></textarea></div><div class="form-group"><label for="email">আপনার ইমেইল (Optional):</label><input type="email" name="email" id="email"></div><input type="hidden" name="reported_content_id" value="{{ prefill_id }}"><button type="submit">Submit</button></form><a href="{{ url_for('home') }}" class="back-link">← Cancel</a>{% endif %}
</div></body></html>
"""

# ======================================================================
# --- Helper Functions (Final Version) ---
# ======================================================================

def parse_filename(filename):
    """
    Final, most robust version of the filename parser.
    It can now detect complete season packs and individual episodes more accurately.
    """
    LANGUAGE_MAP = {
        'hindi': 'Hindi', 'hin': 'Hindi', 'english': 'English', 'eng': 'English',
        'bengali': 'Bengali', 'bangla': 'Bangla', 'ben': 'Bengali',
        'tamil': 'Tamil', 'tam': 'Tamil', 'telugu': 'Telugu', 'tel': 'Telugu',
        'kannada': 'Kannada', 'kan': 'Kannada', 'malayalam': 'Malayalam', 'mal': 'Malayalam',
        'korean': 'Korean', 'kor': 'Korean', 'chinese': 'Chinese', 'chi': 'Chinese',
        'japanese': 'Japanese', 'jap': 'Japanese',
        'dual audio': ['Hindi', 'English'], 'dual': ['Hindi', 'English'],
        'multi audio': ['Multi Audio']
    }
    JUNK_KEYWORDS = [
        '1080p', '720p', '480p', '2160p', '4k', 'uhd', 'web-dl', 'webdl', 'webrip',
        'brrip', 'bluray', 'dvdrip', 'hdrip', 'hdcam', 'camrip', 'hdts', 'x264',
        'x265', 'hevc', 'avc', 'aac', 'ac3', 'dts', '5.1', '7.1', 'final', 'uncut',
        'extended', 'remastered', 'unrated', 'nf', 'www', 'com', 'net', 'org', 'psa'
    ]
    SEASON_PACK_KEYWORDS = ['complete', 'season', 'pack', 'all episodes', 'zip']

    # Remove file extension and normalize separators
    base_name, _ = os.path.splitext(filename)
    processed_name = re.sub(r'[\._\[\]\(\)\{\}-]', ' ', base_name)
    
    # --- Language Detection ---
    found_languages = []
    temp_name_for_lang = processed_name.lower()
    for keyword, lang_name in LANGUAGE_MAP.items():
        if re.search(r'\b' + re.escape(keyword) + r'\b', temp_name_for_lang):
            if isinstance(lang_name, list):
                found_languages.extend(lang_name)
            else:
                found_languages.append(lang_name)
    languages = sorted(list(set(found_languages))) if found_languages else []

    # --- Season Pack Detection ---
    season_pack_match = re.search(r'^(.*?)[\s\.]*(?:S|Season)[\s\.]?(\d{1,2})', processed_name, re.I)
    if season_pack_match:
        text_after_season = processed_name[season_pack_match.end():].lower()
        # A season pack if it contains pack keywords OR doesn't contain episode identifiers
        is_pack = any(keyword in text_after_season for keyword in SEASON_PACK_KEYWORDS) or not re.search(r'\be\d', text_after_season)

        if is_pack:
            title = season_pack_match.group(1).strip()
            season_num = int(season_pack_match.group(2))
            
            for junk in JUNK_KEYWORDS + SEASON_PACK_KEYWORDS:
                title = re.sub(r'\b' + re.escape(junk) + r'\b', '', title, flags=re.I)
            final_title = ' '.join(title.split()).title()
            
            if final_title:
                return {'type': 'series_pack', 'title': final_title, 'season': season_num, 'languages': languages}

    # --- Episodic Detection ---
    series_patterns = [
        re.compile(r'^(.*?)[\s\.]*(?:S|Season)[\s\.]?(\d{1,2})[\s\.]*(?:E|Ep|Episode)[\s\.]?(\d{1,3})', re.I),
        re.compile(r'^(.*?)[\s\.]*(?:E|Ep|Episode)[\s\.]?(\d{1,3})', re.I)
    ]
    for i, pattern in enumerate(series_patterns):
        match = pattern.search(processed_name)
        if match:
            title = match.group(1).strip()
            season_num = int(match.group(2)) if i == 0 else 1
            episode_num = int(match.group(3)) if i == 0 else int(match.group(2))

            for junk in JUNK_KEYWORDS:
                title = re.sub(r'\b' + re.escape(junk) + r'\b', '', title, flags=re.I)
            final_title = ' '.join(title.split()).title()
            
            if final_title:
                return {'type': 'series', 'title': final_title, 'season': season_num, 'episode': episode_num, 'languages': languages}

    # --- Movie Detection (Fallback) ---
    year_match = re.search(r'\b(19[5-9]\d|20\d{2})\b', processed_name)
    year = year_match.group(1) if year_match else None
    title_part = processed_name[:year_match.start()] if year_match else processed_name
    
    temp_title = title_part
    for lang_key in LANGUAGE_MAP.keys():
        temp_title = re.sub(r'\b' + lang_key + r'\b', '', temp_title, flags=re.I)
    for junk in JUNK_KEYWORDS:
        temp_title = re.sub(r'\b' + re.escape(junk) + r'\b', '', temp_title, flags=re.I)
    final_title = ' '.join(temp_title.split()).title()
    
    return {'type': 'movie', 'title': final_title, 'year': year, 'languages': languages} if final_title else None

def get_tmdb_details_from_api(title, content_type, year=None):
    if not TMDB_API_KEY:
        print("ERROR: TMDB_API_KEY is not set.")
        return None
    
    search_type = "tv" if content_type in ["series", "series_pack"] else "movie"
    
    def search_tmdb(query_title):
        print(f"INFO: Searching TMDb for: '{query_title}' (Type: {search_type}, Year: {year})")
        try:
            search_url = f"https://api.themoviedb.org/3/search/{search_type}?api_key={TMDB_API_KEY}&query={requests.utils.quote(query_title)}"
            if year and search_type == "movie":
                search_url += f"&year={year}"
            
            search_res = requests.get(search_url, timeout=10)
            search_res.raise_for_status()
            results = search_res.json().get("results")
            
            if not results: return None
            
            tmdb_id = results[0].get("id")
            detail_url = f"https://api.themoviedb.org/3/{search_type}/{tmdb_id}?api_key={TMDB_API_KEY}&append_to_response=videos"
            detail_res = requests.get(detail_url, timeout=10)
            detail_res.raise_for_status()
            res_json = detail_res.json()

            trailer_key = next((v['key'] for v in res_json.get("videos", {}).get("results", []) if v.get('type') == 'Trailer' and v.get('site') == 'YouTube'), None)
            
            details = {
                "tmdb_id": tmdb_id, "title": res_json.get("title") or res_json.get("name"), 
                "poster": f"https://image.tmdb.org/t/p/w500{res_json.get('poster_path')}" if res_json.get('poster_path') else None, 
                "overview": res_json.get("overview"), "release_date": res_json.get("release_date") or res_json.get("first_air_date"), 
                "genres": [g['name'] for g in res_json.get("genres", [])], "vote_average": res_json.get("vote_average"), "trailer_key": trailer_key
            }
            print(f"SUCCESS: Found TMDb details for '{query_title}' (ID: {tmdb_id}).")
            return details
        except requests.RequestException as e:
            print(f"ERROR: TMDb API request failed for '{query_title}'. Reason: {e}")
            return None

    tmdb_data = search_tmdb(title)
    if not tmdb_data and len(title.split()) > 1:
        simpler_title = " ".join(title.split()[:-1])
        print(f"INFO: Initial search failed. Retrying with simpler title: '{simpler_title}'")
        tmdb_data = search_tmdb(simpler_title)
        
    if not tmdb_data: print(f"WARNING: TMDb search found no results for '{title}' after all attempts.")
    return tmdb_data

def process_movie_list(movie_list):
    return [{**item, '_id': str(item['_id'])} for item in movie_list]

# ======================================================================
# --- Main Flask Routes ---
# ======================================================================

@app.route('/')
def home():
    query = request.args.get('q')
    if query:
        movies_list = list(movies.find({"title": {"$regex": query, "$options": "i"}}).sort('_id', -1))
        return render_template_string(index_html, movies=process_movie_list(movies_list), query=f'Results for "{query}"', is_full_page_list=True)
    all_badges = sorted([badge for badge in movies.distinct("poster_badge") if badge])
    limit = 12
    context = {
        "trending_movies": process_movie_list(list(movies.find({"is_trending": True, "is_coming_soon": {"$ne": True}}).sort('_id', -1).limit(limit))),
        "latest_movies": process_movie_list(list(movies.find({"type": "movie", "is_coming_soon": {"$ne": True}}).sort('_id', -1).limit(limit))),
        "latest_series": process_movie_list(list(movies.find({"type": "series", "is_coming_soon": {"$ne": True}}).sort('_id', -1).limit(limit))),
        "coming_soon_movies": process_movie_list(list(movies.find({"is_coming_soon": True}).sort('_id', -1).limit(limit))),
        "recently_added": process_movie_list(list(movies.find({"is_coming_soon": {"$ne": True}}).sort('_id', -1).limit(6))),
        "recently_added_full": process_movie_list(list(movies.find({"is_coming_soon": {"$ne": True}}).sort('_id', -1).limit(limit))),
        "is_full_page_list": False, "query": "", "all_badges": all_badges
    }
    return render_template_string(index_html, **context)

@app.route('/movie/<movie_id>')
def movie_detail(movie_id):
    try:
        movie = movies.find_one({"_id": ObjectId(movie_id)})
        if not movie: return "Content not found", 404
        related_movies = []
        if movie.get("genres"):
            related_movies = list(movies.find({"genres": {"$in": movie["genres"]}, "_id": {"$ne": ObjectId(movie_id)}}).limit(12))
        return render_template_string(detail_html, movie=movie, trailer_key=movie.get("trailer_key"), related_movies=process_movie_list(related_movies))
    except Exception: return "Content not found", 404

@app.route('/watch/<movie_id>')
def watch_movie(movie_id):
    try:
        movie = movies.find_one({"_id": ObjectId(movie_id)})
        if not movie or not movie.get("watch_link"): return "Content not found.", 404
        return render_template_string(watch_html, watch_link=movie["watch_link"], title=movie["title"])
    except Exception: return "An error occurred.", 500

def render_full_list(content_list, title):
    return render_template_string(index_html, movies=process_movie_list(content_list), query=title, is_full_page_list=True)

@app.route('/badge/<badge_name>')
def movies_by_badge(badge_name): return render_full_list(list(movies.find({"poster_badge": badge_name}).sort('_id', -1)), f'Tag: {badge_name}')

@app.route('/genres')
def genres_page(): return render_template_string(genres_html, genres=sorted([g for g in movies.distinct("genres") if g]), title="Browse by Genre")

@app.route('/genre/<genre_name>')
def movies_by_genre(genre_name): return render_full_list(list(movies.find({"genres": genre_name}).sort('_id', -1)), f'Genre: {genre_name}')

@app.route('/trending_movies')
def trending_movies(): return render_full_list(list(movies.find({"is_trending": True, "is_coming_soon": {"$ne": True}}).sort('_id', -1)), "Trending Now")

@app.route('/movies_only')
def movies_only(): return render_full_list(list(movies.find({"type": "movie", "is_coming_soon": {"$ne": True}}).sort('_id', -1)), "All Movies")

@app.route('/webseries')
def webseries(): return render_full_list(list(movies.find({"type": "series", "is_coming_soon": {"$ne": True}}).sort('_id', -1)), "All Web Series")

@app.route('/coming_soon')
def coming_soon(): return render_full_list(list(movies.find({"is_coming_soon": True}).sort('_id', -1)), "Coming Soon")

@app.route('/recently_added')
def recently_added_all(): return render_full_list(list(movies.find({"is_coming_soon": {"$ne": True}}).sort('_id', -1)), "Recently Added")

# ======================================================================
# --- Admin and Webhook Routes ---
# ======================================================================
@app.route('/admin', methods=["GET", "POST"])
@requires_auth
def admin():
    if request.method == "POST":
        content_type = request.form.get("content_type", "movie")
        tmdb_data = get_tmdb_details_from_api(request.form.get("title"), content_type) or {}
        movie_data = {
            "title": request.form.get("title"), "type": content_type, **tmdb_data, 
            "is_trending": False, "is_coming_soon": False, 
            "links": [], "files": [], "episodes": [], "season_packs": [], "languages": []
        }
        if content_type == "movie":
            # ... (Movie handling code as before)
            pass
        else:
            # ... (Series handling code as before)
            pass
        movies.insert_one(movie_data)
        return redirect(url_for('admin'))

    search_query = request.args.get('search', '').strip()
    query_filter = {}
    if search_query: query_filter = {"title": {"$regex": search_query, "$options": "i"}}
    ad_settings = settings.find_one() or {}
    content_list = process_movie_list(list(movies.find(query_filter).sort('_id', -1)))
    feedback_list = process_movie_list(list(feedback.find().sort('timestamp', -1)))
    return render_template_string(admin_html, content_list=content_list, ad_settings=ad_settings, feedback_list=feedback_list, search_query=search_query)


@app.route('/admin/save_ads', methods=['POST'])
@requires_auth
def save_ads():
    ad_codes = {
        "popunder_code": request.form.get("popunder_code", ""), "social_bar_code": request.form.get("social_bar_code", ""),
        "banner_ad_code": request.form.get("banner_ad_code", ""), "native_banner_code": request.form.get("native_banner_code", "")
    }
    settings.update_one({}, {"$set": ad_codes}, upsert=True)
    return redirect(url_for('admin'))

@app.route('/edit_movie/<movie_id>', methods=["GET", "POST"])
@requires_auth
def edit_movie(movie_id):
    try:
        obj_id = ObjectId(movie_id)
    except Exception:
        return "Invalid Movie ID", 400
    movie_obj = movies.find_one({"_id": obj_id})
    if not movie_obj: return "Movie not found", 404

    if request.method == "POST":
        content_type = request.form.get("content_type", "movie")
        update_data = {
            "title": request.form.get("title"), "type": content_type,
            "is_trending": request.form.get("is_trending") == "true",
            "is_coming_soon": request.form.get("is_coming_soon") == "true",
            "poster": request.form.get("poster", "").strip(), "overview": request.form.get("overview", "").strip(),
            "genres": [g.strip() for g in request.form.get("genres", "").split(',') if g.strip()],
            "languages": [lang.strip() for lang in request.form.get("languages", "").split(',') if lang.strip()],
            "poster_badge": request.form.get("poster_badge", "").strip() or None
        }
        
        if content_type == "movie":
            update_data["watch_link"] = request.form.get("watch_link", "")
            update_data["links"] = [{"quality": q, "url": u} for q, u in [("480p", request.form.get("link_480p")), ("720p", request.form.get("link_720p")), ("1080p", request.form.get("link_1080p"))] if u]
            update_data["files"] = [{"quality": q, "message_id": int(mid)} for q, mid in zip(request.form.getlist('telegram_quality[]'), request.form.getlist('telegram_message_id[]')) if q and mid]
            movies.update_one({"_id": obj_id}, {"$set": update_data, "$unset": {"episodes": "", "season_packs": ""}})
        else:
            update_data["episodes"] = [{"season": int(s), "episode_number": int(e), "title": t, "watch_link": w or None, "message_id": int(m) if m else None} for s, e, t, w, m in zip(request.form.getlist('episode_season[]'), request.form.getlist('episode_number[]'), request.form.getlist('episode_title[]'), request.form.getlist('episode_watch_link[]'), request.form.getlist('episode_message_id[]'))]
            update_data["season_packs"] = [{"season": int(s), "message_id": int(mid)} for s, mid in zip(request.form.getlist('pack_season[]'), request.form.getlist('pack_message_id[]')) if s and mid]
            movies.update_one({"_id": obj_id}, {"$set": update_data, "$unset": {"links": "", "watch_link": "", "files": ""}})
        
        return redirect(url_for('admin'))

    return render_template_string(edit_html, movie=movie_obj)


@app.route('/delete_movie/<movie_id>')
@requires_auth
def delete_movie(movie_id):
    movies.delete_one({"_id": ObjectId(movie_id)})
    return redirect(url_for('admin'))


@app.route('/admin/delete_all_movies')
@requires_auth
def delete_all_movies():
    try:
        result = movies.delete_many({})
        print(f"DELETED: {result.deleted_count} documents from the 'movies' collection by admin.")
    except Exception as e:
        print(f"ERROR: Could not delete all movies. Reason: {e}")
    return redirect(url_for('admin'))


@app.route('/contact', methods=['GET', 'POST'])
def contact():
    if request.method == 'POST':
        feedback_data = {
            "type": request.form.get("type"), "content_title": request.form.get("content_title"),
            "message": request.form.get("message"), "email": request.form.get("email", "").strip(),
            "reported_content_id": request.form.get("reported_content_id"), "timestamp": datetime.utcnow()
        }
        feedback.insert_one(feedback_data)
        return render_template_string(contact_html, message_sent=True)
    prefill_title, prefill_id = request.args.get('title', ''), request.args.get('report_id', '')
    prefill_type = 'Problem Report' if prefill_id else 'Movie Request'
    return render_template_string(contact_html, message_sent=False, prefill_title=prefill_title, prefill_id=prefill_id, prefill_type=prefill_type)


@app.route('/delete_feedback/<feedback_id>')
@requires_auth
def delete_feedback(feedback_id):
    feedback.delete_one({"_id": ObjectId(feedback_id)})
    return redirect(url_for('admin'))


@app.route('/webhook', methods=['POST'])
def telegram_webhook():
    data = request.get_json()
    if 'channel_post' in data:
        post = data['channel_post']
        if str(post.get('chat', {}).get('id')) != ADMIN_CHANNEL_ID: 
            return jsonify(status='ok', reason='not_admin_channel')
        
        file = post.get('video') or post.get('document')
        if not (file and file.get('file_name')): 
            return jsonify(status='ok', reason='no_file_in_post')
        
        filename = file.get('file_name')
        print(f"\n--- [WEBHOOK] PROCESSING NEW FILE: {filename} ---")
        parsed_info = parse_filename(filename)
        
        if not parsed_info or not parsed_info.get('title'):
            print(f"FAILED: Could not parse title from filename: {filename}")
            return jsonify(status='ok', reason='parsing_failed')
        
        print(f"PARSED INFO: {parsed_info}")

        tmdb_data = get_tmdb_details_from_api(parsed_info['title'], parsed_info['type'], parsed_info.get('year'))
        if not tmdb_data or not tmdb_data.get("tmdb_id"):
            print(f"DATABASE: Skipping update for '{parsed_info['title']}' due to no TMDb data.")
            return jsonify(status='ok', reason='no_tmdb_data_or_id')
        
        tmdb_id = tmdb_data.get("tmdb_id")
        update_doc = {"$set": {k: v for k, v in tmdb_data.items() if v}, "$addToSet": {}}
        if parsed_info.get('languages'):
            update_doc["$addToSet"]["languages"] = {"$each": parsed_info['languages']}
        
        content_type = parsed_info['type']
        existing_content = movies.find_one({"tmdb_id": tmdb_id})

        if content_type in ['series', 'series_pack']:
            if not existing_content:
                print(f"DATABASE: No existing series found. Creating new entry for '{tmdb_data['title']}'...")
                series_doc = {**tmdb_data, "type": "series", "episodes": [], "season_packs": [], "languages": parsed_info.get('languages', [])}
                movies.insert_one(series_doc)
                existing_content = movies.find_one({"tmdb_id": tmdb_id}) # Re-fetch the newly created doc
            
            if content_type == 'series_pack':
                new_pack = {"season": parsed_info['season'], "message_id": post['message_id']}
                movies.update_one({"_id": existing_content['_id']}, {"$pull": {"season_packs": {"season": new_pack['season']}}})
                update_doc["$push"] = {"season_packs": new_pack}
                print(f"SUCCESS: Season {new_pack['season']} pack updated.")
            else: # Episodic
                new_episode = {"season": parsed_info['season'], "episode_number": parsed_info['episode'], "message_id": post['message_id']}
                movies.update_one({"_id": existing_content['_id']}, {"$pull": {"episodes": {"season": new_episode['season'], "episode_number": new_episode['episode_number']}}})
                update_doc["$push"] = {"episodes": new_episode}
                print(f"SUCCESS: S{new_episode['season']}E{new_episode['episode_number']} updated.")

            movies.update_one({"_id": existing_content['_id']}, update_doc)

        else: # Movie
            quality_match = re.search(r'(\d{3,4}p)', filename, re.I)
            quality = quality_match.group(1) if quality_match else "HD"
            new_file = {"quality": quality, "message_id": post['message_id']}

            if existing_content:
                movies.update_one({"_id": existing_content['_id']}, {"$pull": {"files": {"quality": new_file['quality']}}})
                update_doc["$push"] = {"files": new_file}
                movies.update_one({"_id": existing_content['_id']}, update_doc)
                print(f"SUCCESS: Movie file ({quality}) updated.")
            else:
                movie_doc = {**tmdb_data, "type": "movie", "files": [new_file], "languages": parsed_info.get('languages', [])}
                movies.insert_one(movie_doc)
                print("SUCCESS: New movie created.")

    elif 'message' in data:
        message = data['message']
        chat_id = message['chat']['id']
        text = message.get('text', '')
        if text.startswith('/start'):
            parts = text.split()
            if len(parts) > 1:
                try:
                    payload_parts = parts[1].split('_')
                    doc_id_str = payload_parts[0]
                    content = movies.find_one({"_id": ObjectId(doc_id_str)})
                    if not content: return jsonify(status='ok')

                    message_to_copy_id = None
                    file_info_text = ""
                    
                    if len(payload_parts) == 2 and payload_parts[1].startswith('S'): # Season Pack
                        season_num = int(payload_parts[1][1:])
                        pack = next((p for p in content.get('season_packs', []) if p.get('season') == season_num), None)
                        if pack:
                            message_to_copy_id = pack.get('message_id')
                            file_info_text = f"Complete Season {season_num}"
                    
                    elif content.get('type') == 'series' and len(payload_parts) == 3: # Episodic
                        s_num, e_num = int(payload_parts[1]), int(payload_parts[2])
                        episode = next((ep for ep in content.get('episodes', []) if ep.get('season') == s_num and ep.get('episode_number') == e_num), None)
                        if episode: 
                            message_to_copy_id = episode.get('message_id')
                            file_info_text = f"S{s_num:02d}E{e_num:02d}"

                    elif content.get('type') == 'movie' and len(payload_parts) == 2: # Movie
                        quality = payload_parts[1]
                        file = next((f for f in content.get('files', []) if f.get('quality') == quality), None)
                        if file: 
                            message_to_copy_id = file.get('message_id')
                            file_info_text = f"({quality})"
                    
                    if message_to_copy_id:
                        caption_text = (
                            f"🎬 *{escape_markdown(content['title'])}* {escape_markdown(file_info_text)}\n\n"
                            f"✅ *Successfully Sent To Your PM*\n\n"
                            f"🔰 Join Our Main Channel\n➡️ [{escape_markdown(BOT_USERNAME)} Main]({MAIN_CHANNEL_LINK})\n\n"
                            f"📢 Join Our Update Channel\n➡️ [{escape_markdown(BOT_USERNAME)} Official]({UPDATE_CHANNEL_LINK})\n\n"
                            f"💬 For Any Help or Request\n➡️ [Contact Developer]({DEVELOPER_USER_LINK})"
                        )
                        payload = {'chat_id': chat_id, 'from_chat_id': ADMIN_CHANNEL_ID, 'message_id': message_to_copy_id, 'caption': caption_text, 'parse_mode': 'MarkdownV2'}
                        res = requests.post(f"{TELEGRAM_API_URL}/copyMessage", json=payload).json()
                        
                        if res.get('ok'):
                            new_msg_id = res['result']['message_id']
                            scheduler.add_job(func=delete_message_after_delay, trigger='date', run_date=datetime.now() + timedelta(minutes=30), args=[chat_id, new_msg_id], id=f'del_{chat_id}_{new_msg_id}', replace_existing=True)
                        else: 
                            requests.get(f"{TELEGRAM_API_URL}/sendMessage", params={'chat_id': chat_id, 'text': "Error sending file. It might have been deleted from the channel."})
                    else: 
                        requests.get(f"{TELEGRAM_API_URL}/sendMessage", params={'chat_id': chat_id, 'text': "Requested file/season not found."})
                except Exception as e:
                    print(f"Error processing /start command: {e}")
                    requests.get(f"{TELEGRAM_API_URL}/sendMessage", params={'chat_id': chat_id, 'text': "An unexpected error occurred."})
            else: 
                welcome_message = (f"👋 Welcome to {BOT_USERNAME}!\n\nBrowse all our content on our website.")
                try:
                    root_url = url_for('home', _external=True)
                    keyboard = {"inline_keyboard": [[{"text": "🎬 Visit Website", "url": root_url}]]}
                    requests.get(f"{TELEGRAM_API_URL}/sendMessage", params={'chat_id': chat_id, 'text': welcome_message, 'reply_markup': str(keyboard).replace("'", '"')})
                except Exception:
                     requests.get(f"{TELEGRAM_API_URL}/sendMessage", params={'chat_id': chat_id, 'text': welcome_message})
    return jsonify(status='ok')


if __name__ == "__main__":
    port = int(os.environ.get("PORT", 5000))
    app.run(host='0.0.0.0', port=port, debug=False)
