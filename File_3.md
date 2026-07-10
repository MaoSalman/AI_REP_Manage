PART 3: CLIENT DASHBOARD (Simple Web App)

Step 4.1: Create Client Dashboard
Create a simple dashboard so clients can see their reviews and approve responses.
bash
mkdir -p /opt/dashboard && cd /opt/dashboard

cat > docker-compose.yml << 'EOF'
version: "3.8"

services:
  dashboard:
    image: nginx:alpine
    restart: always
    ports:
      - "127.0.0.1:8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
EOF

mkdir -p html

cat > html/index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Review Agent Dashboard</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; background: #f5f5f5; color: #333; }
        .container { max-width: 1200px; margin: 0 auto; padding: 20px; }
        header { background: #1a1a2e; color: white; padding: 20px 0; margin-bottom: 30px; }
        header h1 { font-size: 24px; }
        .stats { display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: 20px; margin-bottom: 30px; }
        .stat-card { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
        .stat-card h3 { font-size: 14px; color: #666; margin-bottom: 10px; text-transform: uppercase; }
        .stat-card .number { font-size: 32px; font-weight: bold; color: #1a1a2e; }
        .reviews { background: white; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); overflow: hidden; }
        .reviews-header { padding: 20px; border-bottom: 1px solid #eee; display: flex; justify-content: space-between; align-items: center; }
        .review-item { padding: 20px; border-bottom: 1px solid #eee; }
        .review-item:last-child { border-bottom: none; }
        .review-header { display: flex; justify-content: space-between; margin-bottom: 10px; }
        .reviewer { font-weight: 600; color: #1a1a2e; }
        .rating { color: #f4c430; }
        .review-text { color: #555; margin-bottom: 15px; line-height: 1.6; }
        .response-box { background: #f8f9fa; padding: 15px; border-radius: 6px; border-left: 3px solid #1a1a2e; }
        .response-box label { font-size: 12px; color: #666; text-transform: uppercase; margin-bottom: 5px; display: block; }
        .response-text { color: #333; line-height: 1.6; }
        .actions { margin-top: 15px; display: flex; gap: 10px; }
        .btn { padding: 8px 20px; border: none; border-radius: 6px; cursor: pointer; font-size: 14px; transition: all 0.2s; }
        .btn-approve { background: #28a745; color: white; }
        .btn-approve:hover { background: #218838; }
        .btn-edit { background: #6c757d; color: white; }
        .btn-edit:hover { background: #5a6268; }
        .status-badge { padding: 4px 12px; border-radius: 20px; font-size: 12px; font-weight: 600; }
        .status-pending { background: #fff3cd; color: #856404; }
        .status-approved { background: #d4edda; color: #155724; }
        .status-posted { background: #cce5ff; color: #004085; }
        .login-box { max-width: 400px; margin: 100px auto; background: white; padding: 40px; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); }
        .login-box h2 { margin-bottom: 20px; text-align: center; }
        .form-group { margin-bottom: 20px; }
        .form-group label { display: block; margin-bottom: 5px; font-weight: 500; }
        .form-group input { width: 100%; padding: 10px; border: 1px solid #ddd; border-radius: 6px; font-size: 14px; }
        .btn-login { width: 100%; padding: 12px; background: #1a1a2e; color: white; border: none; border-radius: 6px; font-size: 16px; cursor: pointer; }
        .hidden { display: none; }
    </style>
</head>
<body>
    <div id="login-page">
        <div class="login-box">
            <h2>Review Agent Dashboard</h2>
            <div class="form-group">
                <label>Client ID</label>
                <input type="text" id="clientId" placeholder="Enter your client ID">
            </div>
            <div class="form-group">
                <label>Password</label>
                <input type="password" id="password" placeholder="Enter your password">
            </div>
            <button class="btn-login" onclick="login()">Login</button>
        </div>
    </div>

    <div id="dashboard-page" class="hidden">
        <header>
            <div class="container">
                <h1>Review Agent Dashboard</h1>
            </div>
        </header>
        
        <div class="container">
            <div class="stats">
                <div class="stat-card">
                    <h3>Total Reviews (30 days)</h3>
                    <div class="number" id="total-reviews">0</div>
                </div>
                <div class="stat-card">
                    <h3>Average Rating</h3>
                    <div class="number" id="avg-rating">0.0</div>
                </div>
                <div class="stat-card">
                    <h3>Response Rate</h3>
                    <div class="number" id="response-rate">0%</div>
                </div>
                <div class="stat-card">
                    <h3>Pending Approval</h3>
                    <div class="number" id="pending-count">0</div>
                </div>
            </div>

            <div class="reviews">
                <div class="reviews-header">
                    <h2>Recent Reviews</h2>
                    <select id="filter-status" onchange="filterReviews()">
                        <option value="all">All</option>
                        <option value="pending">Pending Approval</option>
                        <option value="approved">Approved</option>
                        <option value="posted">Posted</option>
                    </select>
                </div>
                <div id="reviews-list"></div>
            </div>
        </div>
    </div>

    <script>
        // Simple client-side auth (replace with proper backend in production)
        const clients = {
            'demo-client': {
                password: 'demo123',
                name: 'Demo Restaurant',
                reviews: [
                    {
                        id: 1,
                        author: 'John Smith',
                        rating: 5,
                        text: 'Amazing food and great service! The pasta was cooked to perfection.',
                        response: 'Thank you so much, John! We are thrilled you enjoyed our pasta. We look forward to welcoming you back soon!',
                        status: 'posted',
                        date: '2026-07-01'
                    },
                    {
                        id: 2,
                        author: 'Sarah Johnson',
                        rating: 2,
                        text: 'Waited 45 minutes for a table even with a reservation. Food was okay but not worth the wait.',
                        response: 'Dear Sarah, we sincerely apologize for the wait time. This is not the experience we strive to provide. Please contact us directly at manager@demo-restaurant.com so we can make this right.',
                        status: 'pending',
                        date: '2026-07-02'
                    }
                ]
            }
        };

        let currentClient = null;

        function login() {
            const id = document.getElementById('clientId').value;
            const pass = document.getElementById('password').value;
            
            if (clients[id] && clients[id].password === pass) {
                currentClient = clients[id];
                document.getElementById('login-page').classList.add('hidden');
                document.getElementById('dashboard-page').classList.remove('hidden');
                loadDashboard();
            } else {
                alert('Invalid credentials');
            }
        }

        function loadDashboard() {
            const reviews = currentClient.reviews;
            document.getElementById('total-reviews').textContent = reviews.length;
            
            const avg = reviews.reduce((sum, r) => sum + r.rating, 0) / reviews.length;
            document.getElementById('avg-rating').textContent = avg.toFixed(1);
            
            const responded = reviews.filter(r => r.status !== 'pending').length;
            document.getElementById('response-rate').textContent = 
                Math.round((responded / reviews.length) * 100) + '%';
            
            const pending = reviews.filter(r => r.status === 'pending').length;
            document.getElementById('pending-count').textContent = pending;
            
            renderReviews(reviews);
        }

        function renderReviews(reviews) {
            const container = document.getElementById('reviews-list');
            container.innerHTML = '';
            
            reviews.forEach(review => {
                const div = document.createElement('div');
                div.className = 'review-item';
                div.innerHTML = `
                    <div class="review-header">
                        <span class="reviewer">${review.author}</span>
                        <span class="rating">${'★'.repeat(review.rating)}${'☆'.repeat(5-review.rating)}</span>
                    </div>
                    <div class="review-text">${review.text}</div>
                    <div class="response-box">
                        <label>AI Suggested Response</label>
                        <div class="response-text">${review.response}</div>
                    </div>
                    <div class="actions">
                        <span class="status-badge status-${review.status}">${review.status}</span>
                        ${review.status === 'pending' ? `
                            <button class="btn btn-approve" onclick="approveReview(${review.id})">Approve & Post</button>
                            <button class="btn btn-edit" onclick="editReview(${review.id})">Edit Response</button>
                        ` : ''}
                    </div>
                `;
                container.appendChild(div);
            });
        }

        function approveReview(id) {
            const review = currentClient.reviews.find(r => r.id === id);
            if (review) {
                review.status = 'posted';
                loadDashboard();
                alert('Response approved and posted to Google!');
            }
        }

        function editReview(id) {
            const review = currentClient.reviews.find(r => r.id === id);
            const newResponse = prompt('Edit response:', review.response);
            if (newResponse) {
                review.response = newResponse;
                renderReviews(currentClient.reviews);
            }
        }

        function filterReviews() {
            const filter = document.getElementById('filter-status').value;
            let reviews = currentClient.reviews;
            if (filter !== 'all') {
                reviews = reviews.filter(r => r.status === filter);
            }
            renderReviews(reviews);
        }
    </script>
</body>
</html>
EOF

docker-compose up -d
Add Nginx config for dashboard:
bash
cat > /etc/nginx/sites-available/dashboard << 'EOF'
server {
    listen 80;
    server_name dashboard.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF

ln -s /etc/nginx/sites-available/dashboard /etc/nginx/sites-enabled/
certbot --nginx -d dashboard.yourdomain.com
