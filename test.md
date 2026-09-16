import UnityPy, zlib, re, os, time, unicodedata
import numpy as np
from PIL import Image, ImageDraw, ImageFilter
from UnityPy.enums import TextureFormat

GOC = r"D:\Downloads\The Adventure Pals\Adventure Pals_Data\resources.assets"
DICH = "resources.assets"
CU = "Good morning, Birthday Boy!"
MOI = "Chào buổi sáng, cậu nhóc sinh nhật!"
VIET = "àáảãạăằắẳẵặâầấẩẫậèéẻẽẹêềếểễệìíỉĩịòóỏõọôồốổỗộơờớởỡợùúủũụưừứửữựỳýỷỹỵđ"
VIET += VIET.upper()

def doc(d):
    raw = d.m_Script
    if isinstance(raw, str): raw = raw.encode("utf-8", "surrogateescape")
    try: return zlib.decompress(raw), True
    except zlib.error: return raw, False

def ghi(d, b, nen):
    if nen: b = zlib.compress(b)
    d.m_Script = b.decode("utf-8", "surrogateescape")
    d.save()
