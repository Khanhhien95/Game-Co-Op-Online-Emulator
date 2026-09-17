import UnityPy, zlib, re, os, unicodedata
import numpy as np
from PIL import Image, ImageDraw, ImageFilter
from UnityPy.enums import TextureFormat

GOC = r"resources.assets"
DICH = "resources.assets.new"
CU = "Good morning, Birthday Boy!"
MOI = "Chào buổi sáng, sinh nhật con trai!"
DX = -3
DY = 0
VIET = "àáảãạăằắẳẵặâầấẩẫậèéẻẽẹêềếểễệìíỉĩịòóỏõọôồốổỗộơờớởỡợùúủũụưừứửữựỳýỷỹỵđ"
VIET += VIET.upper()
WH = (255, 255, 255, 255)


def doc(d):
    raw = d.m_Script
    if isinstance(raw, str):
        raw = raw.encode("utf-8", "surrogateescape")
    try:
        return zlib.decompress(raw), True
    except zlib.error:
        return raw, False


def ghi(d, b, nen):
    if nen:
        b = zlib.compress(b)
    d.m_Script = b.decode("utf-8", "surrogateescape")
    d.save()


# ---------- Nạp file gốc ----------
if not os.path.exists(GOC):
    raise SystemExit("Khong thay file goc: " + GOC)
env = UnityPy.load(GOC)
thoai, fnt_obj, tex_font = [], None, None
texs, xmls = {}, {}
for obj in env.objects:
    if obj.type.name not in ("TextAsset", "Texture2D"):
        continue
    d = obj.read()
    n = d.m_Name
    if obj.type.name == "Texture2D":
        if n == "conversationfont":
            tex_font = d
        else:
            if n in texs:
                print("  Canh bao: trung ten texture, giu ban sau:", n)
            texs[n] = d
    else:
        if n == "conversationfont.fnt":
            fnt_obj = d
        elif n.startswith("endoflevel") and n.endswith(".xml"):
            if n in xmls:
                print("  Canh bao: trung ten xml, giu ban sau:", n)
            xmls[n] = d
        else:
            b, nen = doc(d)
            if CU.encode() in b:
                thoai.append((d, b, nen))

missing = []
if not fnt_obj: missing.append("conversationfont.fnt")
if not tex_font: missing.append("anh conversationfont")
if not xmls: missing.append("cac file endoflevel*.xml")
if not thoai: missing.append(f'doan thoai chua chuoi "{CU}"')
if missing:
    raise SystemExit("Thieu du lieu trong file goc: " + "; ".join(missing))

# ---------- Bước 1: Sửa thoại ----------
for d, b, nen in thoai:
    ghi(d, b.replace(CU.encode(), MOI.encode("utf-8")), nen)
print("Buoc 1 - sua thoai:", len(thoai), "file")

# ---------- Bước 2: Tạo ký tự có dấu ----------
fb, fnen = doc(fnt_obj)
fnt = fb.decode("utf-8")
order = []
for mm in re.findall(r"<char\s[^>]*/>", fnt):
    a = dict(re.findall(r'(\w+)="([^"]*)"', mm))
    order.append({k: int(a[k]) for k in
                  ("id", "x", "y", "width", "height", "xoffset", "yoffset", "xadvance")})
chars = {c["id"]: c for c in order}
if len(order) != 95:
    raise SystemExit("File goc khong sach, dung lai")

atlas_f = tex_font.image.convert("RGBA")


def make(ch):
    nfd = unicodedata.normalize("NFD", ch)
    base, marks = nfd[0], list(nfd[1:])
    if ch in "đĐ":
        base, marks = ("d" if ch == "đ" else "D"), ["BAR"]
    c = chars.get(ord(base))
    if not c:
        return None

    s = c["height"] or 16
    t = max(2, round(s / 7))
    m = s * 0.4
    PT, PB = int(s * 1.1) + 2 * t, int(s * 0.5) + t
    PL = PR = int(s * 0.5)

    g = atlas_f.crop((c["x"], c["y"], c["x"] + c["width"], c["y"] + c["height"]))
    bb = g.getchannel("A").getbbox()
    if not bb:
        return None

    cv = Image.new("RGBA", (g.width + PL + PR, g.height + PT + PB), (0, 0, 0, 0))
    cv.paste(g, (PL, PT))
    dr = ImageDraw.Draw(cv)
    l, top, r, bot = bb[0] + PL, PT + bb[1], bb[2] + PL, PT + bb[3]
    cx, y, extra = (l + r) / 2, top - 1, 0
    co_mu = "\u0302" in marks or "\u0306" in marks
    for mk in marks:
        if mk == "\u031B":
            dr.line([(r - t, top + s * 0.3), (r + m * 0.45, top + s * 0.1),
                     (r + m * 0.45, top - m * 0.3)], fill=WH, width=t, joint="curve")
            extra = int(m * 0.5)
        elif mk == "\u0323":
            rr = max(1.2, t * 0.7)
            dr.ellipse((cx - rr, bot + 1, cx + rr, bot + 1 + 2 * rr), fill=WH)
        elif mk == "BAR":
            if base == "D":
                yy = (top + bot) / 2
                dr.line([(l - t * 0.5, yy), (l + (r - l) * 0.45, yy)], fill=WH, width=t)
            else:
                yy = top + (bot - top) * 0.2
                dr.line([(r - (r - l) * 0.6, yy), (r + t * 0.5, yy)], fill=WH, width=t)
        elif mk == "\u0302":
            dr.line([(cx - m * 0.5, y), (cx, y - m * 0.55), (cx + m * 0.5, y)], fill=WH, width=t)
        elif mk == "\u0306":
            dr.arc((cx - m * 0.5, y - m * 0.8, cx + m * 0.5, y), 0, 180, fill=WH, width=t)
        else:
            tx = cx + (m * 0.6 if co_mu else 0)
            ty = y - (m * 0.15 if co_mu else 0)
            if mk == "\u0301":
                dr.line([(tx - m * 0.2, ty), (tx + m * 0.35, ty - m * 0.7)], fill=WH, width=t)
            elif mk == "\u0300":
                dr.line([(tx - m * 0.35, ty - m * 0.7), (tx + m * 0.2, ty)], fill=WH, width=t)
            elif mk == "\u0303":
                dr.line([(tx - m * 0.55, ty - m * 0.15), (tx - m * 0.2, ty - m * 0.45),
                         (tx + m * 0.2, ty - m * 0.1), (tx + m * 0.55, ty - m * 0.4)],
                        fill=WH, width=t, joint="curve")
            elif mk == "\u0309":
                dr.arc((tx - m * 0.3, ty - m * 0.8, tx + m * 0.3, ty - m * 0.3), 90, 180, fill=WH, width=t)
                dr.line([(tx, ty - m * 0.3), (tx, ty)], fill=WH, width=t)

    box = cv.getchannel("A").getbbox()
    cv = cv.crop(box)
    xo = c["xoffset"] - PL + box[0]
    yo_corrected = c["yoffset"] - (PT - box[1])

    return cv, dict(xoffset=xo, yoffset=yo_corrected,
                    xadvance=c["xadvance"] + c["xoffset"] - xo + extra + DX)


moi = []
for ch in VIET:
    if ord(ch) in chars:
        continue
    res = make(ch)
    if res:
        moi.append((ch, res[0], res[1]))

AW = atlas_f.width
fx, fy0, frowh, font_pos = 1, atlas_f.height + 2, 0, {}
for ch, img, info in moi:
    if fx + img.width + 2 > AW:
        fx, fy0, frowh = 1, fy0 + frowh + 2, 0
    font_pos[ch] = (fx, fy0)
    fx += img.width + 2
    frowh = max(frowh, img.height)
AH = 1
while AH < fy0 + frowh + 2:
    AH *= 2

font_out = Image.new("RGBA", (AW, AH), (0, 0, 0, 0))
font_out.paste(atlas_f, (0, 0))
for ch, img, info in moi:
    font_out.paste(img, font_pos[ch])

lines = ""
for ch, img, info in moi:
    px, py = font_pos[ch]
    lines += (f'<char id="{ord(ch)}" x="{px}" y="{py}" width="{img.width}" height="{img.height}" '
              f'xoffset="{info["xoffset"]}" yoffset="{info["yoffset"] + DY}" '
              f'xadvance="{info["xadvance"]}" page="0" chnl="0" letter="?"/>\n')
fnt = fnt.replace("</chars>", lines + "</chars>")
fnt = re.sub(r'<chars count="\d+"', f'<chars count="{len(order) + len(moi)}"', fnt)
fnt = re.sub(r'scaleW="\d+"', f'scaleW="{AW}"', fnt)
fnt = re.sub(r'scaleH="\d+"', f'scaleH="{AH}"', fnt)
ghi(fnt_obj, fnt.encode("utf-8"), fnen)

# Tối ưu dung lượng Font
try:
    tex_font.set_image(font_out, target_format=TextureFormat.ETC2_RGBA8, mipmap_count=1)
except Exception:
    tex_font.set_image(font_out, target_format=TextureFormat.RGBA32, mipmap_count=1)
tex_font.save()
print("Buoc 2 - them ky tu:", len(moi), "| tong:", len(order) + len(moi),
      "| anh font moi:", AW, "x", AH)

# ---------- Bước 3: Dán hình vào các gói ----------
S = np.array(atlas_f.getchannel("A"), dtype=float)
xong = 0
for xname, xml_obj in sorted(xmls.items()):
    tex = texs.get(xname[:-4])
    xb, xnen = doc(xml_obj)
    xml = xb.decode("utf-8")
    if 'name="conversationfont0"' not in xml:
        continue
    if tex is None:
        print("  Bo qua (khong co anh):", xname)
        continue
    orig_fmt = getattr(tex, "m_TextureFormat", TextureFormat.ETC2_RGBA8)
    imgs = {}
    for tg in re.findall(r"<image\s[^>]*/>", xml):
        a = dict(re.findall(r'(\w+)="([^"]*)"', tg))
        imgs[a["name"]] = a

    atlas = tex.image.convert("RGBA")
    W, H = atlas.size
    A = np.array(atlas.getchannel("A"), dtype=float)
    pwl, phl, sn, sf = [], [], 0, 0
    for i, c in enumerate(order):
        a = imgs.get(f"conversationfont{i}")
        if not a or c["width"] == 0:
            continue
        fl, ft, fw, fh = (int(a[k]) for k in ("frameLeft", "frameTop", "frameWidth", "frameHeight"))
        pwl.append(fw - c["width"])
        phl.append(fh - c["height"])
        g = S[c["y"]:c["y"] + c["height"], c["x"]:c["x"] + c["width"]].sum()
        sn += abs(A[ft:ft + fh, fl:fl + fw].sum() - g)
        sf += abs(A[max(0, H - ft - fh):H - ft, fl:fl + fw].sum() - g)
    pw, ph = int(np.median(pwl)), int(np.median(phl))
    flip = sf < sn
    if flip:
        atlas = atlas.transpose(Image.FLIP_TOP_BOTTOM)

    A2 = np.array(atlas.getchannel("A"), dtype=float)
    votes = {}
    for i, c in enumerate(order):
        a = imgs.get(f"conversationfont{i}")
        if not a or c["width"] == 0:
            continue
        fl, ft = int(a["frameLeft"]), int(a["frameTop"])
        g = S[c["y"]:c["y"] + c["height"], c["x"]:c["x"] + c["width"]]
        best = None
        for oy in range(ph + 1):
            for ox in range(pw + 1):
                v = A2[ft + oy:ft + oy + c["height"], fl + ox:fl + ox + c["width"]]
                if v.shape != g.shape:
                    continue
                dd = abs(v - g).sum()
                if best is None or dd < best[0]:
                    best = (dd, ox, oy)
        if best:
            votes[(best[1], best[2])] = votes.get((best[1], best[2]), 0) + 1
    ox, oy = max(votes, key=votes.get) if votes else (pw // 2, ph // 2)

    occ = np.array(atlas.getchannel("A").point(lambda v: 255 if v else 0)
                   .filter(ImageFilter.MaxFilter(5))) > 0
    them = ""
    for j, (ch, img, info) in enumerate(moi):
        name = f"conversationfont{len(order) + j}"
        fw, fh = img.width + pw, img.height + ph
        I = np.pad(occ, ((1, 0), (1, 0))).astype(np.int32).cumsum(0).cumsum(1)
        sm = I[fh:, fw:] - I[:-fh, fw:] - I[fh:, :-fw] + I[:-fh, :-fw]
        free = np.flatnonzero(sm == 0)
        if len(free) == 0:
            raise SystemExit("Het cho trong: " + xname)
        yy, xx = divmod(int(free[0]), sm.shape[1])
        atlas.paste(img, (xx + ox, yy + oy))
        occ[max(0, yy - 2):yy + fh + 2, max(0, xx - 2):xx + fw + 2] = True
        
        ft_out = H - yy - fh if flip else yy
        them += (f'<image frameTop="{ft_out}" rows="1" frameWidth="{fw}" columnsAndRows="1" '
                 f'name="{name}" centrePointX="0" centrePointY="0" frameLeft="{xx}" '
                 f'frameHeight="{fh}" columns="1"/>')

    ghi(xml_obj, xml.replace("</root>", them + "</root>").encode("utf-8"), xnen)
    if flip:
        atlas = atlas.transpose(Image.FLIP_TOP_BOTTOM)
    
    # Nén lại đúng format nhẹ chuẩn mobile
    try:
        tex.set_image(atlas, target_format=orig_fmt, mipmap_count=1)
    except Exception:
        try:
            tex.set_image(atlas, target_format=TextureFormat.ETC2_RGBA8, mipmap_count=1)
        except Exception:
            tex.set_image(atlas, target_format=TextureFormat.RGBA32, mipmap_count=1)
    tex.save()
    xong += 1
    print("  Da sua:", xname, "| dem:", pw, ph, "| lech:", ox, oy, "| lat:", flip)
print("Buoc 3 - so goi da sua:", xong)

# ---------- Ghi file ----------
open(DICH, "wb").write(env.file.save())
input("XONG. Da luu resources.assets.new. Nhan Enter de thoat...")
