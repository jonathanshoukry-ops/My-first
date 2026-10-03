import json, os, re, shutil, sys, time, uuid, threading, traceback
from http.server import ThreadingHTTPServer, BaseHTTPRequestHandler
from urllib.parse import urlparse, parse_qs
import engine as E

ROOT = os.path.dirname(os.path.abspath(__file__)); DATA = os.path.join(ROOT, 'data'); os.makedirs(DATA, exist_ok=True)
SIZES = {'9:16': (720,1280), '16:9': (1280,720), '1:1': (720,720), '4:5': (720,900)}
VEXT = {'.mp4','.mov','.webm','.mkv'}; AEXT = {'.mp3','.wav','.m4a','.aac'}; MAXB = 1_500_000_000
LOCK = threading.Lock()

def pdir(pid): return os.path.join(DATA, re.sub(r'[^\w-]', '', pid))
def load(pid):
    with open(os.path.join(pdir(pid), 'project.json')) as f: return json.load(f)
def save(p):
    p['modified'] = time.time()
    with LOCK:
        tmp = os.path.join(pdir(p['id']), 'project.tmp')
        with open(tmp, 'w') as f: json.dump(p, f); os.replace(tmp, os.path.join(pdir(p['id']), 'project.json'))

def stage(p, s, err=None):
    p['job'] = dict(stage=s, error=err, running=err is None and s not in ('Completed',)); save(p)

def run(pid, mode, restage=True):
    p = load(pid)
    try:
        ref = next((m for m in p['media'] if m['kind']=='ref'), None); clips = [m for m in p['media'] if m['kind']=='clip' and not m.get('excluded')]
        music = next((m for m in p['media'] if m['kind']=='music'), None)
        if not ref or not clips: raise ValueError('Upload one reference video and at least one clip first.')
        if 'blueprint' not in p:
            stage(p, 'Analyzing Reference'); p['blueprint'] = E.analyze_reference(ref['path']); save(p)
        for c in clips:
            if 'analysis' not in c:
                stage(p, f"Analyzing Footage: {c['name']}"); c['analysis'] = E.analyze_clip(c['path']); save(p)
        stage(p, 'Matching Clips / Building Timeline')
        cl = [dict(id=c['id'], name=c['name'], path=c['path'], **c['analysis']) for c in clips]
        bp = p['blueprint']; mbeats = E.beats(music['path'])['beats'] if music else []
        tl, rep = E.build(bp, cl, mode, p['params'], mbeats, p.get('target'))
        p.setdefault('versions', []).append(dict(ts=time.time(), mode=mode, params=dict(p['params']), timeline=tl, report=rep))
        p['mode'], p['timeline'], p['report'] = mode, tl, rep; save(p)
        W, H = SIZES[p['aspect']]; wd = os.path.join(pdir(pid), 'work'); os.makedirs(wd, exist_ok=True)
        out = os.path.join(pdir(pid), f"export_v{len(p['versions'])}.mp4")
        v = E.render(tl, cl, out, W, H, 30, music['path'] if music else None, wd,
                     lambda s: stage(p, s))
        p['exports'] = p.get('exports', []) + [dict(file=os.path.basename(out), ts=time.time(), **v)]; p['latest'] = os.path.basename(out)
        shutil.rmtree(wd, ignore_errors=True); stage(p, 'Completed')
    except Exception as e:
        traceback.print_exc(); stage(p, 'Failed', f'{type(e).__name__}: {e}'.replace(pdir(pid), '.'))

def modify(p, text):
    t = text.lower(); P = p['params']; up = re.search(r'strong|more|increase|bigger|heav|aggress', t); dn = re.search(r'less|reduc|weak|fewer|smaller|subtle|gentl', t); did = []
    f = 1.5 if up else 0.6 if dn else None
    for k, w in (('shake','shake'), ('zoom','zoom'), ('flash','flash')):
        if w in t and f: P[k] = round(min(P[k]*f, 3), 2); did.append(f'{k} x{f}')
    if 'no flash' in t or 'remove flash' in t: P['flash'] = 0; did.append('flashes removed')
    if 'faster' in t: P['pace'] = round(P['pace']*0.75, 2); did.append('faster pacing (creative mode)')
    if 'slower' in t: P['pace'] = round(P['pace']*1.3, 2); did.append('slower pacing (creative mode)')
    return did

class H(BaseHTTPRequestHandler):
    def log_message(self, *a): pass
    def js(self, o, c=200):
        b = json.dumps(o).encode(); self.send_response(c); self.send_header('Content-Type','application/json'); self.send_header('Content-Length', str(len(b))); self.end_headers(); self.wfile.write(b)
    def body(self):
        n = int(self.headers.get('Content-Length') or 0); return json.loads(self.rfile.read(n) or b'{}')
    def file(self, path, ct, dl=None):
        size = os.path.getsize(path); r = self.headers.get('Range'); a, b = 0, size-1
        if r:
            m = re.match(r'bytes=(\d*)-(\d*)', r); a = int(m[1] or 0); b = int(m[2]) if m[2] else size-1
        self.send_response(206 if r else 200); self.send_header('Content-Type', ct); self.send_header('Accept-Ranges','bytes')
        self.send_header('Content-Length', str(b-a+1))
        if r: self.send_header('Content-Range', f'bytes {a}-{b}/{size}')
        if dl: self.send_header('Content-Disposition', f'attachment; filename="{dl}"')
        self.end_headers()
        with open(path,'rb') as f:
            f.seek(a); left = b-a+1
            while left > 0:
                ch = f.read(min(1<<20, left))
                if not ch: break
                self.wfile.write(ch); left -= len(ch)
    def pub(self, p):
        q = {k: v for k, v in p.items() if k not in ('versions',)}
        q['media'] = [{k: v for k, v in m.items() if k not in ('path','analysis')} | {'analyzed': 'analysis' in m} for m in p['media']]
        q['versionCount'] = len(p.get('versions', [])); return q
    def do_GET(self):
        u = urlparse(self.path); s = u.path.strip('/').split('/')
        try:
            if u.path == '/': return self.file(os.path.join(ROOT, 'index.html'), 'text/html')
            if s[:2] == ['api','projects']:
                ps = [load(d) for d in os.listdir(DATA) if os.path.exists(os.path.join(DATA, d, 'project.json'))]
                return self.js(dict(projects=[self.pub(p) for p in sorted(ps, key=lambda x: -x['modified'])]))
            if s[:2] == ['api','p'] and len(s) == 3: return self.js(self.pub(load(s[2])))
            if s[:2] == ['api','p'] and s[3] in ('video','export','media'):
                p = load(s[2]); q = parse_qs(u.query)
                if s[3] == 'media': m = next(m for m in p['media'] if m['id'] == q['id'][0]); return self.file(m['path'], 'video/mp4' if m['kind'] != 'music' else 'audio/mpeg')
                fn = q.get('f', [p.get('latest')])[0]; fn = os.path.basename(fn)
                return self.file(os.path.join(pdir(s[2]), fn), 'video/mp4', fn if s[3] == 'export' else None)
        except (FileNotFoundError, KeyError, StopIteration, TypeError): return self.js(dict(error='Not found'), 404)
        self.js(dict(error='Not found'), 404)
    def do_POST(self):
        s = urlparse(self.path).path.strip('/').split('/'); b = self.body()
        try:
            if s == ['api','p']:
                pid = uuid.uuid4().hex[:10]; os.makedirs(os.path.join(pdir(pid), 'media'))
                a = b.get('aspect', '9:16'); a = a if a in SIZES else '9:16'
                p = dict(id=pid, name=(b.get('name') or 'Untitled')[:80], aspect=a, created=time.time(), media=[], params=dict(shake=1,zoom=1,flash=1,pace=1), job=dict(stage='Created', running=False, error=None))
                save(p); return self.js(self.pub(p))
            p = load(s[2])
            if s[3] == 'generate':
                if p['job'].get('running'): return self.js(dict(error='A job is already running'), 409)
                p['params'] = dict(shake=1,zoom=1,flash=1,pace=1); p['target'] = b.get('target') or None
                if b.get('reset'): p.pop('blueprint', None)
                stage(p, 'Preparing Media'); threading.Thread(target=run, args=(p['id'], b.get('mode','exact')), daemon=True).start(); return self.js(dict(ok=True))
            if s[3] == 'modify':
                if p['job'].get('running'): return self.js(dict(error='A job is already running'), 409)
                did = modify(p, b.get('text','')); 
                if not did: return self.js(dict(error="Not understood. Supported: stronger/less shake, zoom, flash; remove flashes; faster/slower. Speed/velocity edits aren't implemented."), 400)
                stage(p, 'Building Timeline'); threading.Thread(target=run, args=(p['id'], p['mode']), daemon=True).start(); return self.js(dict(applied=did))
            if s[3] == 'rename': p['name'] = b['name'][:80]; save(p); return self.js(self.pub(p))
            if s[3] == 'clip' and s[4] == 'toggle':
                m = next(m for m in p['media'] if m['id'] == b['id']); m['excluded'] = not m.get('excluded'); save(p); return self.js(self.pub(p))
        except Exception as e: return self.js(dict(error=str(e)), 400)
    def do_PUT(self):
        u = urlparse(self.path); s = u.path.strip('/').split('/'); q = parse_qs(u.query)
        try:
            p = load(s[2]); kind = q['kind'][0]; name = re.sub(r'[^\w.\- ]', '_', q['name'][0])[:80]; ext = os.path.splitext(name)[1].lower()
            n = int(self.headers.get('Content-Length') or 0)
            if kind not in ('ref','clip','music') or ext not in (AEXT if kind=='music' else VEXT): return self.js(dict(error=f'Unsupported file type {ext}'), 400)
            if n > MAXB: return self.js(dict(error='File too large'), 413)
            mid = uuid.uuid4().hex[:8]; path = os.path.join(pdir(p['id']), 'media', mid+ext)
            with open(path, 'wb') as f:
                left = n
                while left > 0:
                    ch = self.rfile.read(min(1<<20, left))
                    if not ch: break
                    f.write(ch); left -= len(ch)
            try: info = E.probe(path); assert kind == 'music' or 'width' in info
            except Exception: os.remove(path); return self.js(dict(error='File is corrupted or not a valid media file'), 400)
            if kind in ('ref','music'): p['media'] = [m for m in p['media'] if m['kind'] != kind]  # one reference / one music track
            if kind == 'ref': p.pop('blueprint', None)
            p['media'].append(dict(id=mid, kind=kind, name=name, path=path, info=info)); save(p); self.js(self.pub(p))
        except Exception as e: self.js(dict(error=str(e)), 400)
    def do_DELETE(self):
        s = urlparse(self.path).path.strip('/').split('/')
        if len(s) == 3: shutil.rmtree(pdir(s[2]), ignore_errors=True)
        self.js(dict(ok=True))

if __name__ == '__main__':
    # projects interrupted by a restart are marked failed so they can be retried
    for d in os.listdir(DATA):
        try:
            p = load(d)
            if p['job'].get('running'): stage(p, 'Failed', 'Interrupted by server restart. Press Generate to retry.')
        except Exception: pass
    port = int(sys.argv[1]) if len(sys.argv) > 1 else 8000
    print(f'RE-EDIT AI on http://localhost:{port}'); ThreadingHTTPServer(('0.0.0.0', port), H).serve_forever()

