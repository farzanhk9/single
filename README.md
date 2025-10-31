#!/usr/bin/env python3
# File: srt_burner.py
# Burn SRT subtitles into a 9:16 video (1080x1920) with cover/blur fit and styled caption boxes.

import argparse, re, os
from typing import List, Tuple, Optional
from PIL import Image, ImageDraw, ImageFont, ImageFilter
from moviepy.editor import VideoFileClip, ImageClip, CompositeVideoClip, concatenate_videoclips, vfx

W, H = 1080, 1920
SAFE_XY = 32  # extra safety inside margins

SRT_RE = re.compile(
    r"(\d+)\s*?\n(\d{2}):(\d{2}):(\d{2}),(\d{3})\s*-->\s*(\d{2}):(\d{2}):(\d{2}),(\d{3})\s*\n(.*?)(?=\n{2,}|\Z)",
    re.DOTALL
)

def parse_hex_color(c: str) -> Tuple[int,int,int]:
    c = c.strip().lstrip("#")
    if len(c)==3: c = "".join(ch*2 for ch in c)
    return tuple(int(c[i:i+2],16) for i in (0,2,4))

def srt_to_entries(text: str) -> List[Tuple[float,float,str]]:
    out = []
    for m in SRT_RE.finditer(text.strip()):
        (_idx,
         h1,m1,s1,ms1,
         h2,m2,s2,ms2,
         content) = (m.group(1),
                     int(m.group(2)), int(m.group(3)), int(m.group(4)), int(m.group(5)),
                     int(m.group(6)), int(m.group(7)), int(m.group(8)), int(m.group(9)),
                     m.group(10).strip())
        start = h1*3600 + m1*60 + s1 + ms1/1000.0
        end   = h2*3600 + m2*60 + s2 + ms2/1000.0
        # Normalize line breaks
        content = re.sub(r"\s*\n\s*", " / ", content).strip()
        out.append((start, end, content))
    return out

def load_font(path: Optional[str], size: int) -> ImageFont.FreeTypeFont:
    if path and os.path.exists(path):
        try: return ImageFont.truetype(path, size=size)
        except Exception: pass
    for cand in ["Arial.ttf", "arial.ttf", "DejaVuSans.ttf"]:
        try: return ImageFont.truetype(cand, size=size)
        except Exception: continue
    return ImageFont.load_default()

def wrap_text(draw: ImageDraw.ImageDraw, text: str, font: ImageFont.ImageFont, max_w: int) -> List[str]:
    words = text.split()
    lines, line = [], ""
    for w in words:
        test = (line + " " + w).strip()
        if draw.textlength(test, font=font) <= max_w:
            line = test
        else:
            if line: lines.append(line)
            line = w
    if line: lines.append(line)
    return lines

def render_caption_image(
    text: str,
    max_width: int,
    font: ImageFont.FreeTypeFont,
    txt_color: Tuple[int,int,int],
    bg_color: Tuple[int,int,int],
    bg_alpha: int,
    pad_x: int = 24,
    pad_y: int = 16,
    shadow: int = 10,
    radius: int = 18,
    line_spacing: float = 1.2,
) -> Image.Image:
    # Measure
    tmp = Image.new("RGBA", (max_width, 10), (0,0,0,0))
    td = ImageDraw.Draw(tmp)
    lines = wrap_text(td, text, font, max_width)
    if not lines: lines = [" "]
    # compute dimensions
    h_list = []
    for ln in lines:
        bbox = font.getbbox(ln)
        h_list.append(bbox[3]-bbox[1])
    line_h = h_list[0] if h_list else font.size
    total_h = int(sum(h_list) + (len(lines)-1) * (line_h*(line_spacing-1)))
    text_w = max(int(td.textlength(ln, font=font)) for ln in lines)
    box_w = text_w + 2*pad_x
    box_h = total_h + 2*pad_y

    # Canvas for box + shadow
    cw, ch = box_w + shadow*2, box_h + shadow*2
    img = Image.new("RGBA", (cw, ch), (0,0,0,0))
    draw = ImageDraw.Draw(img)

    # Shadow
    if shadow > 0:
        sh = Image.new("RGBA", (box_w, box_h), (0,0,0,0))
        sd = ImageDraw.Draw(sh)
        sd.rounded_rectangle([0,0,box_w,box_h], radius=radius, fill=(0,0,0,140))
        sh = sh.filter(ImageFilter.GaussianBlur(shadow/1.8))
        img.alpha_composite(sh, (shadow, shadow))

    # Box
    box = Image.new("RGBA", (box_w, box_h), (0,0,0,0))
    bd = ImageDraw.Draw(box)
    bd.rounded_rectangle([0,0,box_w,box_h], radius=radius, fill=(*bg_color, bg_alpha))
    img.alpha_composite(box, (shadow, shadow))

    # Text
    y = shadow + pad_y
    for ln in lines:
        ln_w = td.textlength(ln, font=font)
        x = shadow + pad_x + (text_w - ln_w)//2
        draw.text((x, y), ln, font=font, fill=txt_color)
        y += int(line_h * line_spacing)
    return img

def fit_video_9x16(clip, mode: str):
    if mode == "cover":
        # scale to fill then crop center
        scale = max(W/clip.w, H/clip.h)
        tmp = clip.resize(scale)
        x = (tmp.w - W) / 2
        y = (tmp.h - H) / 2
        return tmp.crop(x1=x, y1=y, x2=x+W, y2=y+H)
    # blur letterbox: background is blurred full frame; foreground contains
    bg = clip.resize((W,H)).fx(vfx.blur, 30)
    scale = min(W/clip.w, H/clip.h)
    fg = clip.resize(scale).set_position(("center","center"))
    return CompositeVideoClip([bg, fg], size=(W,H))

def make_progress_bar(duration: float, height: int = 10, margin: int = 24, color="#ffffff", alpha=230):
    # returns a function that creates an ImageClip of the progressing bar for given time t
    col = parse_hex_color(color)
    def frame(t):
        w = int((W - 2*margin) * max(0.0, min(1.0, t / max(0.001, duration))))
        img = Image.new("RGBA", (W, H), (0,0,0,0))
        draw = ImageDraw.Draw(img)
        # track background (subtle)
        draw.rounded_rectangle([margin, H - margin - height, W - margin, H - margin],
                               radius=height//2, fill=(255,255,255,80))
        # progress
        draw.rounded_rectangle([margin, H - margin - height, margin + w, H - margin],
                               radius=height//2, fill=(*col, alpha))
        return img
    return frame

def main():
    ap = argparse.ArgumentParser(description="Burn SRT subtitles into a 9:16 (1080x1920) video with styled boxes.")
    ap.add_argument("--video", required=True, help="Input video file")
    ap.add_argument("--srt", required=True, help="Subtitle file (.srt)")
    ap.add_argument("--output", default="reel.mp4")

    ap.add_argument("--fit", choices=["cover","blur"], default="cover", help="cover=crop to fill; blur=contain + blurred bg")
    ap.add_argument("--fps", type=int, default=30)

    ap.add_argument("--font", default=None, help="Path to .ttf/.otf (optional)")
    ap.add_argument("--font-size", type=int, default=54)
    ap.add_argument("--color", default="#ffffff")
    ap.add_argument("--bg-color", default="#111827")
    ap.add_argument("--bg-alpha", type=int, default=170)
    ap.add_argument("--position", choices=["top","middle","bottom"], default="bottom")
    ap.add_argument("--margin", type=int, default=48)
    ap.add_argument("--max-width", type=int, default=920, help="Max text width in px (inside 1080)")
    ap.add_argument("--line-spacing", type=float, default=1.2)
    ap.add_argument("--shadow", type=int, default=10)
    ap.add_argument("--radius", type=int, default=18)

    ap.add_argument("--progress", type=int, default=0, help="Show bottom progress bar (1=yes)")

    args = ap.parse_args()

    # Load video
    base = VideoFileClip(args.video)
    # Normalize rotation if present
    if hasattr(base, "rotation") and base.rotation in (90,270):
        base = base.rotate(-base.rotation)

    fitted = fit_video_9x16(base, args.fit).set_fps(args.fps)

    # Parse SRT
    with open(args.srt, "r", encoding="utf-8", errors="ignore") as f:
        entries = srt_to_entries(f.read())

    # Prepare caption clips
    font = load_font(args.font, args.font_size)
    txt_col = parse_hex_color(args.color)
    bg_col = parse_hex_color(args.bg_color)

    caption_layers = []
    for start, end, content in entries:
        img = render_caption_image(
            content,
            max_width=min(args.max_width, W - 2*(args.margin + SAFE_XY)),
            font=font,
            txt_color=txt_col,
            bg_color=bg_col,
            bg_alpha=args.bg_alpha,
            pad_x=24, pad_y=16,
            shadow=args.shadow,
            radius=args.radius,
            line_spacing=args.line_spacing
        )
        # position
        x = (W - img.width)//2
        if args.position == "top":
            y = args.margin + SAFE_XY
        elif args.position == "middle":
            y = (H - img.height)//2
        else:
            y = H - img.height - args.margin - SAFE_XY

        clip = ImageClip(img).set_start(start).set_end(end).set_position((x,y))
        caption_layers.append(clip)

    layers = [fitted] + caption_layers

    # Optional progress bar
    if args.progress:
        pb_fn = make_progress_bar(fitted.duration, height=10, margin=24, color=args.color, alpha=230)
        # MoviePy compos callback: create frames from function
        prog = (ImageClip(pb_fn(0))
                .set_duration(fitted.duration)
                .fl_image(lambda _frm: _frm)  # placeholder; we’ll update via .fl
               )
        # Time-dynamic frame function
        prog = prog.fl(lambda gf, t: pb_fn(t))
        layers.append(prog)

    final = CompositeVideoClip(layers, size=(W,H))
    # Keep original audio
    final = final.set_audio(fitted.audio)

    final.write_videofile(
        args.output,
        fps=args.fps,
        codec="libx264",
        audio_codec="aac",
        preset="medium",
        threads=2,
        bitrate="6M"
    )
    print(f"✅ Done. Saved to {args.output}")

if __name__ == "__main__":
    main()

