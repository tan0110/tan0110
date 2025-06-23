# Video Automation System - Deployment Guide

## Quick Start Deployment

### Option 1: Docker Compose (Recommended)

1. **Prerequisites**
   ```bash
   # Install Docker and Docker Compose
   sudo apt update
   sudo apt install docker.io docker-compose
   sudo usermod -aG docker $USER
   # Log out and back in for group changes to take effect
   ```

2. **Clone and Setup**
   ```bash
   git clone <repository-url>
   cd video_automation_system
   cp .env.example .env
   ```

3. **Configure Environment**
   Edit `.env` file with your settings:
   ```bash
   # Required for YouTube uploads
   YOUTUBE_API_KEY=your_youtube_api_key
   GOOGLE_CLIENT_ID=your_google_client_id
   GOOGLE_CLIENT_SECRET=your_google_client_secret
   
   # Optional for TikTok uploads
   TIKTOK_SESSION_ID=your_tiktok_session
   
   # System configuration
   TARGET_NICHE=personal finance
   VIDEOS_PER_DAY=3
   DAILY_RUN_TIME=08:00
   ```

4. **Deploy**
   ```bash
   cd docker
   docker-compose up -d
   ```

5. **Initialize Database**
   ```bash
   docker-compose exec video-automation python scripts/setup_database.py
   ```

6. **Test System**
   ```bash
   docker-compose exec video-automation python scripts/daily_runner.py --mode test
   ```

### Option 2: Local Development

1. **Install Dependencies**
   ```bash
   # Python 3.11+
   pip install -r requirements.txt
   python -m spacy download en_core_web_sm
   
   # PostgreSQL and Redis
   sudo apt install postgresql redis-server
   ```

2. **Setup Database**
   ```bash
   sudo -u postgres createdb video_automation
   sudo -u postgres createuser video_user
   sudo -u postgres psql -c "ALTER USER video_user WITH PASSWORD 'video_password';"
   sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE video_automation TO video_user;"
   ```

3. **Configure Environment**
   ```bash
   cp .env.example .env
   # Edit .env with local database URL:
   # DATABASE_URL=postgresql://video_user:video_password@localhost:5432/video_automation
   ```

4. **Initialize and Test**
   ```bash
   python scripts/setup_database.py
   python scripts/daily_runner.py --mode test
   ```

## API Setup Instructions

### YouTube Data API

1. **Google Cloud Console Setup**
   - Go to [Google Cloud Console](https://console.cloud.google.com/)
   - Create new project or select existing
   - Enable "YouTube Data API v3"

2. **Create Credentials**
   - Go to "Credentials" → "Create Credentials"
   - Create API Key (for basic access)
   - Create OAuth 2.0 Client ID (for uploads)
   - Download `credentials.json`

3. **Configure OAuth**
   - Set authorized redirect URIs: `http://localhost:8080/`
   - Download and place `credentials.json` in project root

### Google Trends (pytrends)

No API key required - uses unofficial library `pytrends`

### TikTok Upload (Optional)

1. **Browser Method**
   - Log into TikTok in Chrome/Firefox
   - Export cookies using browser extension
   - Save as `cookies.txt` in project root

2. **Session ID Method**
   - Extract `sessionid` cookie value
   - Add to `.env` as `TIKTOK_SESSION_ID`

## Production Deployment

### Cloud Deployment (AWS/GCP/Azure)

1. **Infrastructure Requirements**
   - 4+ CPU cores
   - 8GB+ RAM
   - 50GB+ storage
   - GPU instance (optional, for faster video generation)

2. **Docker Deployment**
   ```bash
   # Build and push to registry
   docker build -t video-automation:latest -f docker/Dockerfile .
   docker tag video-automation:latest your-registry/video-automation:latest
   docker push your-registry/video-automation:latest
   
   # Deploy with docker-compose
   docker-compose -f docker/docker-compose.yml up -d
   ```

3. **Kubernetes Deployment**
   ```yaml
   # kubernetes/deployment.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: video-automation
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: video-automation
     template:
       metadata:
         labels:
           app: video-automation
       spec:
         containers:
         - name: video-automation
           image: your-registry/video-automation:latest
           env:
           - name: DATABASE_URL
             valueFrom:
               secretKeyRef:
                 name: video-automation-secrets
                 key: database-url
   ```

### Monitoring and Logging

1. **Application Logs**
   ```bash
   # Docker logs
   docker-compose logs -f video-automation
   
   # Local logs
   tail -f logs/app.log
   ```

2. **Database Monitoring**
   ```sql
   -- Check recent videos
   SELECT * FROM videos ORDER BY created_at DESC LIMIT 10;
   
   -- Check system logs
   SELECT * FROM system_logs WHERE level = 'ERROR' ORDER BY timestamp DESC;
   ```

3. **Performance Metrics**
   - Monitor CPU/RAM usage during video generation
   - Track API quota usage (YouTube, etc.)
   - Monitor disk space for video storage

### Backup and Recovery

1. **Database Backup**
   ```bash
   # PostgreSQL backup
   pg_dump video_automation > backup_$(date +%Y%m%d).sql
   
   # Restore
   psql video_automation < backup_20231201.sql
   ```

2. **Video Files Backup**
   ```bash
   # Sync to cloud storage
   aws s3 sync data/videos/ s3://your-bucket/videos/
   ```

## Scaling Considerations

### Horizontal Scaling

1. **Message Queue Integration**
   ```python
   # Add to requirements.txt
   celery==5.3.0
   
   # Configure Celery for async processing
   # app/celery_app.py
   from celery import Celery
   
   celery_app = Celery('video_automation')
   celery_app.config_from_object('app.celery_config')
   ```

2. **Load Balancing**
   ```yaml
   # docker-compose.yml
   services:
     video-automation-1:
       build: .
       environment:
         - WORKER_ID=1
     
     video-automation-2:
       build: .
       environment:
         - WORKER_ID=2
     
     nginx:
       image: nginx
       ports:
         - "80:80"
   ```

### Multi-Niche Support

1. **Configuration**
   ```bash
   # .env
   TARGET_NICHES=personal finance,tech gadgets,health tips
   VIDEOS_PER_NICHE=2
   ```

2. **Database Schema**
   ```sql
   -- Add niche-specific configurations
   CREATE TABLE niche_configs (
       id SERIAL PRIMARY KEY,
       niche VARCHAR(100) NOT NULL,
       keywords JSON,
       posting_schedule JSON,
       active BOOLEAN DEFAULT TRUE
   );
   ```

## Troubleshooting

### Common Issues

1. **Database Connection Errors**
   ```bash
   # Check PostgreSQL status
   sudo systemctl status postgresql
   
   # Check connection
   psql -h localhost -U video_user -d video_automation
   ```

2. **YouTube Upload Failures**
   ```bash
   # Check API quotas
   # Verify credentials.json exists
   # Check OAuth token expiration
   ```

3. **Memory Issues**
   ```bash
   # Monitor memory usage
   docker stats
   
   # Increase Docker memory limits
   # docker-compose.yml
   services:
     video-automation:
       deploy:
         resources:
           limits:
             memory: 8G
   ```

4. **Model Loading Errors**
   ```bash
   # Clear model cache
   rm -rf ~/.cache/huggingface/
   
   # Download models manually
   python -c "from transformers import pipeline; pipeline('text-generation', model='microsoft/DialoGPT-medium')"
   ```

### Performance Optimization

1. **GPU Acceleration**
   ```yaml
   # docker-compose.yml
   services:
     video-automation:
       runtime: nvidia
       environment:
         - NVIDIA_VISIBLE_DEVICES=all
   ```

2. **Model Optimization**
   ```python
   # Use quantized models
   model = AutoModelForCausalLM.from_pretrained(
       "microsoft/DialoGPT-medium",
       torch_dtype=torch.float16,
       device_map="auto"
   )
   ```

3. **Caching Strategy**
   ```python
   # Cache generated content
   @lru_cache(maxsize=100)
   def generate_script(topic, keywords):
       # Implementation
       pass
   ```

## Security Considerations

1. **API Key Management**
   ```bash
   # Use environment variables
   # Never commit API keys to version control
   # Rotate keys regularly
   ```

2. **Database Security**
   ```sql
   -- Create read-only user for monitoring
   CREATE USER monitor_user WITH PASSWORD 'secure_password';
   GRANT SELECT ON ALL TABLES IN SCHEMA public TO monitor_user;
   ```

3. **Network Security**
   ```yaml
   # docker-compose.yml
   services:
     postgres:
       networks:
         - internal
       # Don't expose to external network
   
   networks:
     internal:
       driver: bridge
   ```

## Maintenance

### Regular Tasks

1. **Daily**
   - Check system logs for errors
   - Monitor video generation success rate
   - Verify social media uploads

2. **Weekly**
   - Review performance metrics
   - Update trending keywords
   - Check API quota usage

3. **Monthly**
   - Update dependencies
   - Backup database
   - Review and optimize content templates

### Updates and Upgrades

1. **Application Updates**
   ```bash
   # Pull latest code
   git pull origin main
   
   # Rebuild containers
   docker-compose build
   docker-compose up -d
   ```

2. **Dependency Updates**
   ```bash
   # Update requirements
   pip-compile requirements.in
   
   # Test in staging environment first
   ```

This deployment guide provides comprehensive instructions for setting up and maintaining the Video Automation System in various environments.

