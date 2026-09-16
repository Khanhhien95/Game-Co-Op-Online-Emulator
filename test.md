import UnityPy, re, zlib, os

for f in ["resources.assets", "sharedassets0.assets", "level0", "globalgamemanagers.assets"]:
    if not os.path.exists(f):
        continue
    env = UnityPy.load(f)
    for obj in env.objects:
        if obj.type.name != "TextAsset":
            continue
        d = obj.read()
        if not d.m_Name.endswith(".xml"):
            continue
        b = d.m_Script
        if isinstance(b, str): b = b.encode("utf-8", "surrogateescape")
        try: b = zlib.decompress(b)
        except zlib.error: pass
        x = b.decode("utf-8", "ignore")
        n = len(re.findall(r'name="conversationfont\d+"', x))
        if n:
            print(f, "|", d.m_Name, "| PathID", obj.path_id, "| so hinh conversationfont:", n)

input("Xong. Nhan Enter de thoat...")
