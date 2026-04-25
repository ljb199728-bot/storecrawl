"""
스토어크롤 - 스마트스토어 크롤러
TAB 1: 스토어 URL → 상품 링크 수집
"""
import asyncio
import re
import json
from datetime import datetime
from pathlib import Path
from typing import Optional

import httpx
from fastapi import FastAPI, BackgroundTasks
from fastapi.responses import HTMLResponse, JSONResponse, FileResponse
from fastapi.staticfiles import StaticFiles
from pydantic import BaseModel
import uvicorn

app = FastAPI(title="StoreCrawl")

# 작업 상태 저장 (메모리)
jobs: dict = {}

# 정적 파일 (HTML 등)
BASE_DIR = Path(__file__).parent


# ─────────────────────────────────────
# 모델
# ─────────────────────────────────────
class CrawlRequest(BaseModel):
    url: str


# ─────────────────────────────────────
# 메인 페이지
# ─────────────────────────────────────
@app.get("/", response_class=HTMLResponse)
async def index():
    html_path = BASE_DIR / "static" / "index.html"
    return HTMLResponse(html_path.read_text(encoding="utf-8"))


# ─────────────────────────────────────
# 크롤링 시작
# ─────────────────────────────────────
@app.post("/api/crawl/start")
async def start_crawl(req: CrawlRequest, background_tasks: BackgroundTasks):
    url = req.url.strip().rstrip("/")
    
    # 카테고리 경로 제거 → 스토어 기본 URL로 정규화
    if "/category/" in url:
        url = url.split("/category/")[0]
    if "?" in url:
        url = url.split("?")[0]
    
    if not url.startswith("https://smartstore.naver.com/"):
        return JSONResponse(
            {"error": "스마트스토어 URL이 아닙니다. (예: https://smartstore.naver.com/스토어명)"},
            status_code=400
        )
    
    job_id = datetime.now().strftime("%Y%m%d%H%M%S")
    jobs[job_id] = {
        "status": "running",
        "progress": 0,
        "total": 0,
        "message": "스토어 분석 중...",
        "store_name": url.split("/")[-1],
        "urls": [],
        "started_at": datetime.now().isoformat(),
    }
    
    background_tasks.add_task(crawl_store_links, job_id, url)
    return {"job_id": job_id, "store_name": jobs[job_id]["store_name"]}


# ─────────────────────────────────────
# 작업 상태 조회 (실시간 진행상황)
# ─────────────────────────────────────
@app.get("/api/crawl/status/{job_id}")
async def get_status(job_id: str):
    if job_id not in jobs:
        return JSONResponse({"error": "작업을 찾을 수 없습니다."}, status_code=404)
    
    job = jobs[job_id]
    return {
        "status": job["status"],
        "progress": job["progress"],
        "total": job["total"],
        "message": job["message"],
        "store_name": job["store_name"],
        "url_count": len(job.get("urls", [])),
    }


# ─────────────────────────────────────
# 결과 다운로드 (txt 파일)
# ─────────────────────────────────────
@app.get("/api/crawl/download/{job_id}")
async def download_result(job_id: str):
    if job_id not in jobs or jobs[job_id]["status"] != "done":
        return JSONResponse({"error": "결과 없음"}, status_code=404)
    
    job = jobs[job_id]
    
    # 하이크롤과 동일한 포맷으로 txt 생성
    content_lines = [
        f"# 스토어: {job['store_name']}",
        f"# 수집일시: {job['started_at']}",
        f"# 총 상품수: {len(job['urls'])}",
        "",
    ]
    content_lines.extend(job["urls"])
    
    content = "\n".join(content_lines)
    
    # 임시 파일로 저장
    output_path = BASE_DIR / f"output_{job_id}.txt"
    output_path.write_text(content, encoding="utf-8")
    
    return FileResponse(
        str(output_path),
        filename=f"{job['store_name']}.txt",
        media_type="text/plain"
    )


# ─────────────────────────────────────
# 결과 미리보기 (URL 목록)
# ─────────────────────────────────────
@app.get("/api/crawl/preview/{job_id}")
async def preview_result(job_id: str, limit: int = 50):
    if job_id not in jobs:
        return JSONResponse({"error": "결과 없음"}, status_code=404)
    
    job = jobs[job_id]
    return {
        "store_name": job["store_name"],
        "total": len(job.get("urls", [])),
        "urls": job.get("urls", [])[:limit],
    }


# ─────────────────────────────────────
# 핵심: 상품 링크 수집 로직
# ─────────────────────────────────────
async def crawl_store_links(job_id: str, store_url: str):
    """스마트스토어 전체 상품 링크 수집"""
    job = jobs[job_id]
    
    try:
        # 1. channelNo 추출 (스토어 고유번호)
        job["message"] = "스토어 정보 분석 중..."
        channel_no = await get_channel_no(store_url)
        
        if not channel_no:
            job["status"] = "error"
            job["message"] = "스토어 정보를 가져올 수 없습니다. URL을 확인해주세요."
            return
        
        job["channel_no"] = channel_no
        job["message"] = f"상품 목록 조회 중... (채널: {channel_no})"
        
        # 2. 전체 상품 목록 API로 조회
        all_urls = await fetch_all_product_urls(channel_no, store_url, job)
        
        if not all_urls:
            job["status"] = "error"
            job["message"] = "수집된 상품이 없습니다."
            return
        
        # 3. 완료 처리
        job["urls"] = all_urls
        job["status"] = "done"
        job["progress"] = len(all_urls)
        job["total"] = len(all_urls)
        job["message"] = f"수집 완료! 총 {len(all_urls):,}개 상품"
    
    except Exception as e:
        job["status"] = "error"
        job["message"] = f"오류 발생: {str(e)}"


async def get_channel_no(store_url: str) -> Optional[str]:
    """스토어 URL에서 channelNo 추출"""
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
        "Accept-Language": "ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7",
    }
    
    async with httpx.AsyncClient(timeout=15, follow_redirects=True) as client:
        try:
            resp = await client.get(store_url, headers=headers)
            html = resp.text
            
            # 여러 패턴으로 channelNo 추출 시도
            patterns = [
                r'"channelNo"\s*:\s*"?(\d+)"?',
                r'channelNo["\']?\s*[:=]\s*["\']?(\d+)',
                r'/stores/(\d+)/',
                r'channel-no["\']?\s*[:=]\s*["\']?(\d+)',
            ]
            
            for pattern in patterns:
                match = re.search(pattern, html)
                if match:
                    return match.group(1)
            
            return None
        
        except Exception as e:
            print(f"channelNo 추출 실패: {e}")
            return None


async def fetch_all_product_urls(channel_no: str, store_url: str, job: dict) -> list:
    """내부 API로 전체 상품 URL 수집"""
    all_urls = []
    page = 1
    page_size = 80  # 한 페이지당 최대 80개
    total_count = None
    
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/120.0.0.0 Safari/537.36",
        "Accept": "application/json",
        "Accept-Language": "ko-KR,ko;q=0.9",
        "Referer": store_url,
    }
    
    api_base = f"https://smartstore.naver.com/i/v1/stores/{channel_no}/categories/ALL/products"
    
    async with httpx.AsyncClient(timeout=30) as client:
        while True:
            params = {
                "page": page,
                "pageSize": page_size,
                "sortType": "POPULAR",
                "includeOutOfStocks": "true",
                "useStaticPrice": "false",
            }
            
            try:
                resp = await client.get(api_base, params=params, headers=headers)
                if resp.status_code != 200:
                    print(f"API 응답 오류: {resp.status_code}")
                    break
                
                data = resp.json()
                
                # 첫 페이지에서 전체 개수 확인
                if total_count is None:
                    total_count = data.get("totalCount", 0)
                    job["total"] = total_count
                
                items = data.get("simpleProducts", [])
                if not items:
                    break
                
                # 상품 URL 생성
                for item in items:
                    product_no = item.get("productNo")
                    if product_no:
                        url = f"{store_url}/products/{product_no}"
                        all_urls.append(url)
                
                # 진행상황 업데이트
                job["progress"] = len(all_urls)
                job["message"] = f"수집 중... {len(all_urls):,} / {total_count:,}"
                
                # 모두 수집했으면 종료
                if len(all_urls) >= total_count:
                    break
                
                page += 1
                # 안전 딜레이 (네이버 서버 부담 방지)
                await asyncio.sleep(0.3)
            
            except Exception as e:
                print(f"페이지 {page} 수집 실패: {e}")
                # 1번 더 재시도
                await asyncio.sleep(2)
                try:
                    resp = await client.get(api_base, params=params, headers=headers)
                    data = resp.json()
                    items = data.get("simpleProducts", [])
                    for item in items:
                        product_no = item.get("productNo")
                        if product_no:
                            all_urls.append(f"{store_url}/products/{product_no}")
                    page += 1
                except:
                    break
    
    return all_urls


# ─────────────────────────────────────
# 정적 파일 서빙
# ─────────────────────────────────────
static_dir = BASE_DIR / "static"
static_dir.mkdir(exist_ok=True)
app.mount("/static", StaticFiles(directory=str(static_dir)), name="static")


# ─────────────────────────────────────
# 서버 실행
# ─────────────────────────────────────
if __name__ == "__main__":
    import os
    port = int(os.environ.get("PORT", 8000))
    uvicorn.run("main:app", host="0.0.0.0", port=port, reload=False)
