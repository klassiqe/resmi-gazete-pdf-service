from flask import Flask, request, jsonify
import requests
from PyPDF2 import PdfReader
import threading
import uuid
import io
import json
from datetime import datetime
import os

app = Flask(__name__)
# İş durumlarını saklamak için basit bir sözlük
job_storage = {}

@app.route('/process-pdf', methods=['POST'])
def process_pdf():
    try:
        data = request.get_json()
        pdf_url = data.get('pdf_url', '')
        # Eğer job_id gönderilmemişse, benzersiz bir ID oluştur
        job_id = data.get('job_id', str(uuid.uuid4()))
        
        # Asenkron işlemi başlat
        threading.Thread(target=process_pdf_async, args=(pdf_url, job_id)).start()
        
        return jsonify({
            'status': 'processing',
            'job_id': job_id,
            'message': 'PDF işleme başlatıldı. Sonucu almak için /result/job_id adresini kullanın.'
        })
    except Exception as e:
        return jsonify({'error': str(e)}), 500

def process_pdf_async(pdf_url, job_id):
    try:
        # User-Agent ekleyerek bazı sitelerin engellemesini aş
        headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'}
        # PDF'i indir
        response = requests.get(pdf_url, timeout=30, headers=headers)
        response.raise_for_status() # HTTP hatalarını yakala (örn. 404)
        
        pdf_file = io.BytesIO(response.content)
        reader = PdfReader(pdf_file)
        
        text = ""
        for page in reader.pages:
            # Metin çıkarma sırasında None gelebilir, bunu kontrol et
            page_text = page.extract_text()
            if page_text:
                text += page_text
        
        # Birden fazla boşluğu tek boşluğa indirge (Metni temizle)
        text = " ".join(text.split())
        
        result = {
            'status': 'completed',
            # Çok uzun metinleri kısalt (200,000 karakter sınırı)
            'text': text[:200000],
            'page_count': len(reader.pages),
            'processed_at': datetime.now().isoformat()
        }
        
        # İşlem tamamlandığında sonucu sakla
        job_storage[job_id] = result
        
    except Exception as e:
        # Hata oluştuğunda durumu sakla
        job_storage[job_id] = {
            'status': 'error',
            'error': str(e),
            'processed_at': datetime.now().isoformat()
        }

@app.route('/result/<job_id>')
def get_result(job_id):
    result = job_storage.get(job_id)
    if not result:
        # Eğer iş kimliği bulunamazsa 404 döndür
        return jsonify({'status': 'not_found', 'message': f'Job ID ({job_id}) bulunamadı.'}), 404
    return jsonify(result)

@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'service': 'resmi-gazete-processor'})

@app.route('/')
def home():
    return jsonify({'service': 'Resmî Gazete PDF İşleyici', 'status': 'ready', 'endpoints': ['/process-pdf (POST)', '/result/<job_id> (GET)']})

if __name__ == '__main__':
    # Railway/Heroku gibi platformlar PORT ortam değişkenini kullanır
    port = int(os.environ.get('PORT', 5000))
    app.run(host='0.0.0.0', port=port)
