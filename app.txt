/* यशोदा गृहचव — storefront logic (vanilla JS, no build step) */
(function(){
  "use strict";

  /* ---------- Config ---------- */
  var PHONE = "919049803588";
  var MIN = 500, STEP = 500, MAX = 50000;          // grams
  var PCS_MIN = 100, PCS_STEP = 100;                // pieces
  var STORAGE_KEY = "yashoda-gruhchav-cart-v3";
  var IMG = "assets/img/";

  var DATA = [], RATES = {}, ITEM_IMG = {}, BEST = [], SPECIAL = null;
  /* Offer: whole-order discount by total kg. Loaded from offers.json (admin-editable). */
  var OFFER = {enabled:false, tiers:[]};
  var cart = {};
  var ALL = "सर्व";
  var activeCategory = ALL, searchTerm = "";

  var $ = function(id){ return document.getElementById(id); };
  function on(el, ev, fn){ if(el) el.addEventListener(ev, fn); }
  function el(tag, cls, text){
    var n = document.createElement(tag);
    if(cls) n.className = cls;
    if(text != null) n.textContent = text;
    return n;
  }
  function bust(url){ return url + (url.indexOf("?") < 0 ? "?" : "&") + "v=" + Date.now(); }

  /* ---------- Pricing helpers ---------- */
  function rateFor(group, name){
    var r = RATES[group] && RATES[group][name];
    if(typeof r === "number") return {amount:r, basis:"किलो", quantityUnit:"g"};
    if(r && typeof r.amount === "number") return r;
    return null;
  }
  function unitFor(group, name){
    var r = rateFor(group, name);
    return r && r.quantityUnit === "pieces" ? "pieces" : "g";
  }
  function fmtKg(kg){ return (Math.round(kg * 10) / 10).toString(); }
  function fmtQty(q, unit){
    if(unit === "pieces") return q + " नग";
    if(q < 1000) return q + " ग्रॅम";
    return fmtKg(q / 1000) + " किलो";
  }
  function money(n){
    var whole = Math.abs(n - Math.round(n)) < 0.005;
    return "₹" + n.toLocaleString("en-IN", {minimumFractionDigits: whole ? 0 : 2, maximumFractionDigits: 2});
  }
  function round2(n){ return Math.round(n * 100) / 100; }

  function cleanTiers(list){
    var seen = {};
    return (Array.isArray(list) ? list : []).map(function(t){
      return {minKg: Number(t && t.minKg), pct: Number(t && t.pct)};
    }).filter(function(t){
      if(!(t.minKg > 0 && t.minKg <= 1000 && t.pct > 0 && t.pct <= 50) || seen[t.minKg]) return false;
      seen[t.minKg] = 1; return true;
    }).sort(function(a, b){ return b.minKg - a.minKg; });
  }
  function offerOn(){ return OFFER.enabled && OFFER.tiers.length > 0; }
  /* % for an order of `kg` total */
  function orderPct(kg){
    if(!offerOn()) return 0;
    for(var i = 0; i < OFFER.tiers.length; i++){ if(kg >= OFFER.tiers[i].minKg) return OFFER.tiers[i].pct; }
    return 0;
  }
  function nextOrderTier(kg){
    if(!offerOn()) return null;
    var best = null;
    OFFER.tiers.forEach(function(t){ if(t.minKg > kg && (!best || t.minKg < best.minKg)) best = t; });
    return best ? {needKg: round2(best.minKg - kg), pct: best.pct} : null;
  }
  function lineCost(item){
    var r = rateFor(item.group, item.name);
    if(!r) return null;
    return round2(r.amount * item.g / (r.quantityUnit === "pieces" ? 100 : 1000));
  }
  function imgFor(group, name){
    var key = group + "|" + name;
    if(ITEM_IMG[key]) return IMG + ITEM_IMG[key] + ".webp";
    var c = DATA.filter(function(x){ return x.t === group; })[0];
    return IMG + (c && c.img ? c.img : "papad") + ".webp";
  }

  /* ---------- Offer text everywhere on the page ---------- */
  function tiersAsc(){ return OFFER.tiers.slice().sort(function(a, b){ return a.minKg - b.minKg; }); }
  function applyOfferUI(){
    var live = offerOn();
    Array.prototype.forEach.call(document.querySelectorAll("[data-offer]"), function(n){ n.hidden = !live; });
    if(!live) return;
    var asc = tiersAsc();
    var phrase = asc.map(function(t){ return fmtKg(t.minKg) + " किलोपासून " + t.pct + "%"; }).join(" · ");
    Array.prototype.forEach.call(document.querySelectorAll(".js-offer-tick"), function(n){
      n.textContent = "🎉 एकूण ऑर्डरवर सूट: " + phrase;
    });
    Array.prototype.forEach.call(document.querySelectorAll(".js-offer-short"), function(n){
      n.textContent = " एकूण " + fmtKg(asc[0].minKg) + " किलोपासून सूट.";
    });
    Array.prototype.forEach.call(document.querySelectorAll(".js-offer-faq"), function(n){
      n.textContent = "संपूर्ण ऑर्डरचे एकूण वजन " + asc.map(function(t){
        return fmtKg(t.minKg) + " किलो किंवा जास्त झाल्यास " + t.pct + "%";
      }).join(", ") + " सूट एकूण रकमेवर आपोआप लागू होते. पार्सल खर्चावर सूट नाही.";
    });
    var ol = $("tiers");
    if(ol){
      ol.innerHTML = "";
      asc.forEach(function(t, i){
        var li = el("li", i === asc.length - 1 ? "top" : null);
        li.appendChild(el("b", null, t.pct + "%"));
        li.appendChild(el("span", null, "एकूण " + fmtKg(t.minKg) + " किलोपासून"));
        ol.appendChild(li);
      });
    }
  }

  /* ---------- Storage ---------- */
  function saveCart(){ try{ localStorage.setItem(STORAGE_KEY, JSON.stringify(cart)); }catch(e){} }
  function loadCart(){
    try{
      var saved = JSON.parse(localStorage.getItem(STORAGE_KEY) || "{}");
      if(!saved || typeof saved !== "object") return;
      Object.keys(saved).forEach(function(k){
        var it = saved[k];
        if(!it || typeof it.name !== "string" || typeof it.group !== "string" || typeof it.g !== "number") return;
        if(!DATA.some(function(c){ return c.t === it.group && c.i.indexOf(it.name) !== -1; })) return;
        var unit = unitFor(it.group, it.name);
        var mn = unit === "pieces" ? PCS_MIN : MIN, st = unit === "pieces" ? PCS_STEP : STEP;
        var g = Math.round(it.g / st) * st;
        if(g >= mn && g <= MAX) cart[k] = {group: it.group, name: it.name, g: g, unit: unit};
      });
    }catch(e){}
  }

  /* ---------- Toast ---------- */
  function toast(msg){
    var t = $("toast"); if(!t) return;
    t.textContent = msg; t.classList.add("on");
    clearTimeout(toast.timer);
    toast.timer = setTimeout(function(){ t.classList.remove("on"); }, 2000);
  }

  /* ---------- Cart mutations ---------- */
  function addItem(group, name){
    var before = totals().pct;
    var key = group + "|" + name, unit = unitFor(group, name);
    cart[key] = {group: group, name: name, g: unit === "pieces" ? PCS_MIN : MIN, unit: unit};
    refresh(key, "plus");
    var after = totals().pct;
    toast(after > before ? "🎉 एकूण ऑर्डरवर " + after + "% सूट लागू झाली!" : name + " ऑर्डरमध्ये जोडले");
    var b = $("cartBadge"); if(b){ b.classList.remove("bump"); void b.offsetWidth; b.classList.add("bump"); }
  }
  function changeQty(key, dir){
    var it = cart[key]; if(!it) return;
    var mn = it.unit === "pieces" ? PCS_MIN : MIN, st = it.unit === "pieces" ? PCS_STEP : STEP;
    var before = totals().pct;
    if(dir < 0){
      if(it.g <= mn){ delete cart[key]; toast(it.name + " ऑर्डरमधून काढले"); refresh(key, "add"); return; }
      it.g -= st;
    } else {
      if(it.g >= MAX){ toast(it.unit === "pieces" ? "कमाल प्रमाण गाठले" : "कमाल वजन 50 किलो आहे"); return; }
      it.g += st;
    }
    var after = totals().pct;
    if(after > before) toast("🎉 एकूण ऑर्डरवर " + after + "% सूट लागू झाली!");
    refresh(key, dir < 0 ? "minus" : "plus");
  }
  function removeItem(key){
    var it = cart[key]; if(!it) return;
    delete cart[key]; toast(it.name + " ऑर्डरमधून काढले"); refresh();
  }

  /* ---------- Product card ---------- */
  function card(group, name, opts){
    var key = group + "|" + name, item = cart[key], r = rateFor(group, name);
    var unit = unitFor(group, name);
    var c = el("article", "card" + (item ? " on" : ""));
    c.setAttribute("data-key", key);

    var fig = el("div", "card-img");
    var im = el("img"); im.src = imgFor(group, name); im.alt = name; im.loading = "lazy"; im.width = 800; im.height = 600;
    fig.appendChild(im);
    if(opts && opts.tag) fig.appendChild(el("span", "card-tag", group));
    c.appendChild(fig);

    var body = el("div", "card-body");
    body.appendChild(el("h3", null, name));
    if(r){
      var pr = el("div", "price", "₹" + r.amount.toLocaleString("en-IN") + " ");
      pr.appendChild(el("small", null, "/ " + r.basis));
      body.appendChild(pr);
    } else {
      body.appendChild(el("div", "price ask", "दर WhatsApp वर विचारा"));
    }

    var act = el("div", "card-act");
    if(!item){
      var b = el("button", "add", "+ जोडा");
      b.type = "button"; b.setAttribute("data-act", "add");
      b.setAttribute("aria-label", name + " जोडा, किमान " + (unit === "pieces" ? "100 नग" : "500 ग्रॅम"));
      b.addEventListener("click", function(){ addItem(group, name); });
      act.appendChild(b);
    } else {
      var s = el("div", "step");
      var m = el("button", null, "−"); m.type = "button"; m.setAttribute("data-act", "minus"); m.setAttribute("aria-label", name + " कमी करा");
      var o = el("output", null, fmtQty(item.g, item.unit));
      var pl = el("button", null, "+"); pl.type = "button"; pl.setAttribute("data-act", "plus"); pl.setAttribute("aria-label", name + " वाढवा");
      m.addEventListener("click", function(){ changeQty(key, -1); });
      pl.addEventListener("click", function(){ changeQty(key, 1); });
      s.appendChild(m); s.appendChild(o); s.appendChild(pl);
      act.appendChild(s);
    }
    body.appendChild(act);
    c.appendChild(body);
    return c;
  }

  /* ---------- Sections ---------- */
  function buildCategories(){
    var g = $("catGrid"), fc = $("footCats");
    if(g) g.innerHTML = "";
    if(fc) fc.innerHTML = "";
    DATA.forEach(function(cat){
      if(g){
        var b = el("button", "cat-tile"); b.type = "button";
        var im = el("img"); im.src = IMG + (cat.img || "papad") + ".webp"; im.alt = ""; im.loading = "lazy"; im.width = 800; im.height = 600;
        var sp = el("span");
        sp.appendChild(el("b", null, cat.t));
        if(cat.d) sp.appendChild(el("small", null, cat.d));
        sp.appendChild(el("em", null, cat.i.length + " प्रकार"));
        b.appendChild(im); b.appendChild(sp);
        b.addEventListener("click", function(){ pickCategory(cat.t); });
        g.appendChild(b);
      }
      if(fc){
        var li = el("li"), a = el("button", null, cat.t); a.type = "button";
        a.addEventListener("click", function(){ pickCategory(cat.t); });
        li.appendChild(a); fc.appendChild(li);
      }
    });
  }
  function pickCategory(name){
    activeCategory = name; searchTerm = "";
    var s = $("productSearch"); if(s) s.value = "";
    buildFilters(); renderBoard();
    var m = $("menu"); if(m) m.scrollIntoView({behavior:"smooth", block:"start"});
  }
  function buildFilters(){
    var wrap = $("filters"); if(!wrap) return;
    wrap.innerHTML = "";
    [ALL].concat(DATA.map(function(c){ return c.t; })).forEach(function(name){
      var b = el("button", "chip" + (name === activeCategory ? " active" : ""), name);
      b.type = "button"; b.setAttribute("aria-pressed", String(name === activeCategory));
      b.addEventListener("click", function(){ activeCategory = name; buildFilters(); renderBoard(); });
      wrap.appendChild(b);
    });
  }
  function renderBest(){
    var rail = $("bestRail"); if(!rail) return;
    rail.innerHTML = "";
    BEST.forEach(function(k){
      var p = k.split("|"); if(p.length !== 2) return;
      if(!DATA.some(function(c){ return c.t === p[0] && c.i.indexOf(p[1]) !== -1; })) return;
      rail.appendChild(card(p[0], p[1], {tag:true}));
    });
  }
  function renderBoard(){
    var board = $("board"); if(!board) return;
    board.innerHTML = "";
    var q = searchTerm.toLowerCase(), shown = 0;
    DATA.forEach(function(cat){
      if(activeCategory !== ALL && activeCategory !== cat.t) return;
      var items = cat.i.filter(function(n){ return !q || n.toLowerCase().indexOf(q) !== -1 || cat.t.toLowerCase().indexOf(q) !== -1; });
      if(!items.length) return;
      var lab = el("h3", "cat-label", cat.t); lab.appendChild(el("small", null, items.length + " प्रकार"));
      board.appendChild(lab);
      items.forEach(function(n){ board.appendChild(card(cat.t, n)); shown++; });
    });
    if(!shown){
      var e = el("div", "empty-search");
      e.innerHTML = "<strong>पदार्थ सापडला नाही.</strong><br>दुसरे नाव किंवा श्रेणी निवडून पुन्हा प्रयत्न करा.";
      board.appendChild(e);
    }
  }

  /* ---------- Totals (discount on the final amount, by total kg) ---------- */
  function delivery(){
    var r = document.querySelector('input[name="delivery"]:checked');
    return r ? r.value : "local";
  }
  function totals(){
    var t = {g:0, pcs:0, sub:0, pct:0, off:0, net:0, unpriced:0, count:0};
    Object.keys(cart).forEach(function(k){
      var it = cart[k]; t.count++;
      if(it.unit === "pieces") t.pcs += it.g; else t.g += it.g;
      var c = lineCost(it);
      if(c === null){ t.unpriced++; return; }
      t.sub += c;
    });
    t.sub = round2(t.sub);
    t.kg = t.g / 1000;
    t.pct = orderPct(t.kg);
    t.off = round2(t.sub * t.pct / 100);
    t.net = round2(t.sub - t.off);
    t.next = nextOrderTier(t.kg);
    return t;
  }
  function qtyText(t){
    var a = [];
    if(t.g) a.push(fmtQty(t.g, "g"));
    if(t.pcs) a.push(fmtQty(t.pcs, "pieces"));
    return a.join(" + ") || "0";
  }
  function hintText(t){
    if(!t.next) return t.pct ? "🎉 एकूण ऑर्डरवर " + t.pct + "% सूट लागू" : "";
    return (t.pct ? t.pct + "% सूट लागू · " : "") + "अजून " + fmtKg(t.next.needKg) + " किलो घ्या → " + t.next.pct + "% सूट";
  }
  function summary(){
    var keys = Object.keys(cart), lines = $("lines"), t = totals(), has = keys.length > 0;
    if(lines){
      lines.innerHTML = "";
      keys.forEach(function(k){
        var it = cart[k], c = lineCost(it), li = el("li"), r = rateFor(it.group, it.name);
        var im = el("img"); im.src = imgFor(it.group, it.name); im.alt = ""; im.width = 56; im.height = 56; im.loading = "lazy";
        var nm = el("div", "l-name", it.name);
        nm.appendChild(el("small", null, it.group + (r ? " · " + money(r.amount) + " / " + r.basis : "")));

        var right = el("div", "l-right");
        right.appendChild(el("div", "l-cost", c === null ? "दर कळवला जाईल" : money(c)));
        var ms = el("div", "mini-step");
        var m = el("button", null, "−"); m.type = "button"; m.setAttribute("aria-label", it.name + " कमी करा");
        var o = el("output", null, fmtQty(it.g, it.unit));
        var p = el("button", null, "+"); p.type = "button"; p.setAttribute("aria-label", it.name + " वाढवा");
        m.addEventListener("click", function(){ changeQty(k, -1); });
        p.addEventListener("click", function(){ changeQty(k, 1); });
        ms.appendChild(m); ms.appendChild(o); ms.appendChild(p);
        var rm = el("button", "rm", "काढा"); rm.type = "button"; rm.setAttribute("aria-label", it.name + " काढून टाका");
        rm.addEventListener("click", function(){ removeItem(k); });
        right.appendChild(ms); right.appendChild(rm);

        li.appendChild(im); li.appendChild(nm); li.appendChild(right);
        lines.appendChild(li);
      });
      lines.hidden = !has;
    }
    var set = function(id, v){ var n = $(id); if(n) n.textContent = v; };
    var show = function(id, v){ var n = $(id); if(n) n.hidden = !v; };
    show("empty", !has); show("totals", has);
    set("totalw", qtyText(t)); set("subTotal", money(t.sub));
    set("saveLabel", "सूट " + t.pct + "% (एकूण " + fmtKg(t.kg) + " किलो) 🎉");
    set("saveTotal", "−" + money(t.off)); show("saveRow", t.off > 0);
    var hint = hintText(t); set("offerHint", hint); show("offerHint", has && !!hint);
    set("grandTotal", money(t.net));
    show("unpricedNote", t.unpriced > 0);
    show("parcelNote", delivery() === "parcel");
    set("cartCount", t.count + " पदार्थ");
    set("cartBadge", t.count);
    set("bc", t.count); set("bt", money(t.net));
    set("bs", hint || "ऑर्डर 3 ते 4 दिवसांत मिळेल");
    var bar = $("bar"); if(bar) bar.classList.toggle("on", has);
    document.body.classList.toggle("has-cart", has);

    var send = $("send");
    if(send){
      send.setAttribute("aria-disabled", has ? "false" : "true");
      send.href = has ? buildLink(t) : "#";
    }
  }

  function buildLink(t){
    var groups = {};
    Object.keys(cart).forEach(function(k){ var it = cart[k]; (groups[it.group] = groups[it.group] || []).push(it); });
    var m = "नमस्कार, मला यशोदा गृहचव कडून खालील ऑर्डर द्यायची आहे:\n";
    Object.keys(groups).forEach(function(g){
      m += "\n*" + g + "*\n";
      groups[g].forEach(function(it){
        var c = lineCost(it);
        m += "• " + it.name + " – " + fmtQty(it.g, it.unit) + " – " + (c === null ? "दर कळवावा" : money(c)) + "\n";
      });
    });
    m += "\nएकूण प्रमाण: " + qtyText(t) + "\n";
    m += "एकूण रक्कम: " + money(t.sub) + "\n";
    if(t.off > 0) m += "सूट " + t.pct + "% (एकूण " + fmtKg(t.kg) + " किलो): −" + money(t.off) + "\n";
    m += "*देय रक्कम: " + money(t.net) + "*\n";
    if(t.unpriced) m += "दर नसलेले पदार्थ: " + t.unpriced + " (किंमत कळवावी)\n";

    var v = function(id){ var n = $(id); return n ? n.value.trim() : ""; };
    var parcel = delivery() === "parcel";
    m += "\nनाव: " + v("fName") + "\n";
    m += "मोबाईल: " + v("fPhone") + "\n";
    m += "डिलिव्हरी: " + (parcel ? "बाहेरगावी पार्सल" : "छत्रपती संभाजीनगरमध्ये") + "\n";
    if(v("fAddr")) m += "पत्ता: " + v("fAddr") + "\n";
    if(parcel && v("fPin")) m += "पिनकोड: " + v("fPin") + "\n";
    if(v("fNote")) m += "सूचना: " + v("fNote") + "\n";
    if(parcel) m += "\nकृपया पार्सल खर्च कळवा.";
    m += "\nऑर्डर 3 ते 4 दिवसांत मिळेल हे मला माहीत आहे. धन्यवाद!";
    return "https://wa.me/" + PHONE + "?text=" + encodeURIComponent(m);
  }

  /* Re-render and keep keyboard focus on the same product control */
  function refresh(focusKey, prefer){
    saveCart(); renderBest(); renderBoard(); summary();
    if(!focusKey) return;
    var sel = '[data-key="' + focusKey.replace(/"/g, '\\"') + '"]';
    var host = document.activeElement && document.activeElement.closest ? document.activeElement.closest(".rail,.grid") : null;
    var scope = host || $("board");
    var cardEl = scope ? scope.querySelector(sel) : null;
    if(!cardEl) return;
    var btn = cardEl.querySelector('[data-act="' + (prefer || "plus") + '"]') || cardEl.querySelector("button");
    if(btn) btn.focus({preventScroll:true});
  }

  /* ---------- Form validation & send ---------- */
  function markBad(id, bad){ var n = $(id); if(n) n.classList.toggle("bad", !!bad); }
  on($("send"), "click", function(e){
    var st = $("formStatus");
    var say = function(msg){ if(st) st.textContent = msg; };
    if(this.getAttribute("aria-disabled") === "true"){
      e.preventDefault(); say("कृपया आधी किमान एक पदार्थ निवडा.");
      var m = $("menu"); if(m) m.scrollIntoView({behavior:"smooth"}); return;
    }
    var name = $("fName").value.trim(), phone = $("fPhone").value.trim();
    var parcel = delivery() === "parcel", addr = $("fAddr").value.trim(), pin = $("fPin").value.trim();
    markBad("fName", !name); markBad("fPhone", !/^[6-9]\d{9}$/.test(phone));
    markBad("fAddr", parcel && !addr); markBad("fPin", parcel && !/^\d{6}$/.test(pin));
    if(!name){ e.preventDefault(); say("कृपया तुमचे नाव भरा."); $("fName").focus(); return; }
    if(!/^[6-9]\d{9}$/.test(phone)){ e.preventDefault(); say("कृपया 10 अंकी मोबाईल नंबर भरा."); $("fPhone").focus(); return; }
    if(parcel && !addr){ e.preventDefault(); say("पार्सलसाठी पूर्ण पत्ता भरा."); $("fAddr").focus(); return; }
    if(parcel && !/^\d{6}$/.test(pin)){ e.preventDefault(); say("पार्सलसाठी 6 अंकी पिनकोड भरा."); $("fPin").focus(); return; }
    this.href = buildLink(totals());
    say("WhatsApp उघडत आहे… तिथे \"Send\" दाबा.");
  });
  ["fName","fPhone","fAddr","fPin","fNote"].forEach(function(id){
    on($(id), "input", function(){
      if(id === "fPhone" || id === "fPin") this.value = this.value.replace(/\D/g, "");
      this.classList.remove("bad"); summary();
    });
  });
  Array.prototype.forEach.call(document.querySelectorAll('input[name="delivery"]'), function(r){
    r.addEventListener("change", function(){
      var pw = $("pinWrap"); if(pw) pw.hidden = delivery() !== "parcel";
      summary();
    });
  });
  on($("orderForm"), "submit", function(e){ e.preventDefault(); });

  /* ---------- Search ---------- */
  on($("productSearch"), "input", function(){
    searchTerm = this.value.trim();
    if(searchTerm && activeCategory !== ALL){ activeCategory = ALL; buildFilters(); }
    renderBoard();
  });

  /* ---------- Mobile nav ---------- */
  var menuBtn = $("menuBtn"), mobileNav = $("mobileNav");
  function setMenu(open){
    if(!menuBtn || !mobileNav) return;
    mobileNav.hidden = !open;
    menuBtn.setAttribute("aria-expanded", String(open));
    menuBtn.setAttribute("aria-label", open ? "मेनू बंद करा" : "मेनू उघडा");
    menuBtn.textContent = open ? "×" : "☰";
  }
  on(menuBtn, "click", function(){ setMenu(mobileNav.hidden); });
  if(mobileNav) Array.prototype.forEach.call(mobileNav.querySelectorAll("a"), function(a){ a.addEventListener("click", function(){ setMenu(false); }); });

  /* ---------- Special (flyer) ---------- */
  function setupSpecial(){
    var name = SPECIAL && SPECIAL.name ? SPECIAL.name : "खापरावरची पुरणपोळी (मांडे)";
    var price = $("specialPrice");
    if(price){
      if(SPECIAL && typeof SPECIAL.amount === "number"){
        price.innerHTML = "";
        price.appendChild(document.createTextNode(money(SPECIAL.amount) + " "));
        price.appendChild(el("small", null, "/ " + (SPECIAL.basis || "नग")));
      } else price.textContent = "दर विचारा";
    }
    var b = $("specialBtn");
    if(b) b.href = "https://wa.me/" + PHONE + "?text=" + encodeURIComponent(
      "नमस्कार, मला " + name + " ची आगाऊ ऑर्डर द्यायची आहे.\nनग: \nकोणत्या दिवशी हवी: \nनाव: \nपत्ता: ");
  }

  /* Hide the floating WhatsApp button while the footer is on screen (keeps the corner free) */
  (function(){
    var f = document.querySelector(".footer");
    if(!f || !("IntersectionObserver" in window)) return;
    new IntersectionObserver(function(es){ document.body.classList.toggle("footer-in", es[0].isIntersecting); }, {threshold:0.15}).observe(f);
  })();

  var yr = $("year"); if(yr) yr.textContent = new Date().getFullYear();

  /* =========================================================
     ADMIN — offers editor
     • Double-tap / double-click the invisible spot at the bottom-right of the footer.
     • Password never stored. It decrypts a GitHub token kept encrypted in admin.json
       (PBKDF2-SHA256 + AES-GCM). The token then saves offers.json to the GitHub repo,
       and GitHub Pages publishes it for every visitor within about a minute.
     ========================================================= */
  var ADMIN = {cfg:null, token:null, owner:"", repo:"", branch:"main"};
  var ITER = 310000;
  var enc = window.TextEncoder ? new TextEncoder() : null, dec = window.TextDecoder ? new TextDecoder() : null;
  function b64(buf){ var s = "", a = new Uint8Array(buf); for(var i = 0; i < a.length; i++) s += String.fromCharCode(a[i]); return btoa(s); }
  function unb64(str){ var s = atob(str), a = new Uint8Array(s.length); for(var i = 0; i < s.length; i++) a[i] = s.charCodeAt(i); return a; }
  function b64utf8(str){ return b64(enc.encode(str)); }
  function cryptoOK(){ return !!(enc && dec && window.crypto && crypto.subtle && window.isSecureContext); }
  function deriveKey(pass, salt, iter){
    return crypto.subtle.importKey("raw", enc.encode(pass), "PBKDF2", false, ["deriveKey"]).then(function(base){
      return crypto.subtle.deriveKey({name:"PBKDF2", salt:salt, iterations:iter, hash:"SHA-256"}, base,
        {name:"AES-GCM", length:256}, false, ["encrypt", "decrypt"]);
    });
  }
  function encryptToken(token, pass){
    var salt = crypto.getRandomValues(new Uint8Array(16)), iv = crypto.getRandomValues(new Uint8Array(12));
    return deriveKey(pass, salt, ITER).then(function(key){
      return crypto.subtle.encrypt({name:"AES-GCM", iv:iv}, key, enc.encode(token));
    }).then(function(ct){ return {salt:b64(salt), iv:b64(iv), data:b64(ct), iter:ITER}; });
  }
  function decryptToken(cfg, pass){
    return deriveKey(pass, unb64(cfg.salt), cfg.iter || ITER).then(function(key){
      return crypto.subtle.decrypt({name:"AES-GCM", iv:unb64(cfg.iv)}, key, unb64(cfg.data));
    }).then(function(pt){ return dec.decode(pt); });
  }
  function gh(path, opts, token, owner, repo){
    opts = opts || {};
    var h = {"Accept":"application/vnd.github+json", "X-GitHub-Api-Version":"2022-11-28", "Authorization":"Bearer " + token};
    if(opts.body) h["Content-Type"] = "application/json";
    return fetch("https://api.github.com/repos/" + encodeURIComponent(owner) + "/" + encodeURIComponent(repo) + path,
      {method:opts.method || "GET", headers:h, body:opts.body, cache:"no-store"});
  }
  function ghPutFile(file, text, message, token, owner, repo, branch){
    var p = "/contents/" + file;
    return gh(p + "?ref=" + encodeURIComponent(branch), null, token, owner, repo).then(function(r){
      if(r.status === 404) return null;
      if(!r.ok) throw new Error("read " + r.status);
      return r.json().then(function(j){ return j.sha; });
    }).then(function(sha){
      var body = {message:message, content:b64utf8(text), branch:branch};
      if(sha) body.sha = sha;
      return gh(p, {method:"PUT", body:JSON.stringify(body)}, token, owner, repo);
    }).then(function(r){
      if(!r.ok) return r.text().then(function(t){ throw new Error("save " + r.status + " " + t.slice(0, 120)); });
      return true;
    });
  }

  var modal = $("adminModal"), fab = $("adminFab");
  function view(name){
    ["adminLogin","adminSetup","adminEditor"].forEach(function(id){ var n = $(id); if(n) n.hidden = id !== name; });
    var t = $("adminTitle");
    if(t) t.textContent = name === "adminSetup" ? "ॲडमिन सेटअप" : name === "adminEditor" ? "ऑफर सेटिंग्ज" : "ॲडमिन लॉगिन";
    var f = $(name) && $(name).querySelector("input"); if(f) setTimeout(function(){ f.focus(); }, 50);
  }
  function msg(id, text, ok){ var n = $(id); if(n){ n.textContent = text; n.classList.toggle("ok", !!ok); } }
  function openAdmin(){
    if(!modal) return;
    modal.hidden = false; document.body.classList.add("admin-open");
    if(ADMIN.token){ fillEditor(); view("adminEditor"); return; }
    fetch(bust("admin.json"), {cache:"no-store"}).then(function(r){ return r.ok ? r.json() : null; })
      .catch(function(){ return null; })
      .then(function(cfg){
        ADMIN.cfg = cfg && cfg.data ? cfg : null;
        if(!cryptoOK()){ view("adminLogin"); msg("admLoginMsg", "हे फक्त https वेबसाइटवर (GitHub Pages) चालते."); return; }
        if(ADMIN.cfg) { view("adminLogin"); msg("admLoginMsg", ""); }
        else { view("adminSetup"); msg("admSetupMsg", "अजून सेटअप झालेले नाही. एकदा सेटअप करा."); }
      });
  }
  function closeAdmin(){ if(modal){ modal.hidden = true; document.body.classList.remove("admin-open"); } }

  /* Hidden trigger: double-click (desktop) or two quick taps (mobile) */
  (function(){
    var spot = $("adminHotspot"), last = 0;
    if(!spot) return;
    spot.addEventListener("click", function(){
      var now = Date.now();
      if(now - last < 450){ last = 0; openAdmin(); } else last = now;
    });
  })();
  on(fab, "click", openAdmin);
  on($("adminClose"), "click", closeAdmin);
  on(modal, "click", function(e){ if(e.target === modal) closeAdmin(); });
  document.addEventListener("keydown", function(e){ if(e.key === "Escape" && modal && !modal.hidden) closeAdmin(); });
  on($("admToSetup"), "click", function(){
    view("adminSetup"); msg("admSetupMsg", "");
    if(ADMIN.cfg){ $("setOwner").value = ADMIN.cfg.owner || ""; $("setRepo").value = ADMIN.cfg.repo || "yashoda-gruhchav"; $("setBranch").value = ADMIN.cfg.branch || "main"; }
  });
  on($("admToLogin"), "click", function(){ view("adminLogin"); });

  on($("adminLogin"), "submit", function(e){
    e.preventDefault();
    var pass = $("admPass").value;
    if(!ADMIN.cfg){ msg("admLoginMsg", "अजून सेटअप झालेले नाही."); return; }
    msg("admLoginMsg", "तपासत आहे…");
    decryptToken(ADMIN.cfg, pass).then(function(token){
      ADMIN.token = token; ADMIN.owner = ADMIN.cfg.owner; ADMIN.repo = ADMIN.cfg.repo; ADMIN.branch = ADMIN.cfg.branch || "main";
      $("admPass").value = ""; msg("admLoginMsg", "");
      if(fab) fab.hidden = false;
      fillEditor(); view("adminEditor");
    }).catch(function(){ msg("admLoginMsg", "पासवर्ड चुकीचा आहे."); $("admPass").select(); });
  });

  on($("adminSetup"), "submit", function(e){
    e.preventDefault();
    var owner = $("setOwner").value.trim(), repo = $("setRepo").value.trim(), branch = $("setBranch").value.trim() || "main";
    var token = $("setToken").value.trim(), p1 = $("setPass").value, p2 = $("setPass2").value;
    if(!owner || !repo || !token){ msg("admSetupMsg", "सर्व माहिती भरा."); return; }
    if(p1.length < 8){ msg("admSetupMsg", "पासवर्ड किमान 8 अक्षरांचा हवा."); return; }
    if(p1 !== p2){ msg("admSetupMsg", "दोन्ही पासवर्ड जुळत नाहीत."); return; }
    if(!cryptoOK()){ msg("admSetupMsg", "हे फक्त https वेबसाइटवर (GitHub Pages) चालते."); return; }
    msg("admSetupMsg", "GitHub तपासत आहे…");
    gh("", null, token, owner, repo).then(function(r){
      if(r.status === 401) throw new Error("token चुकीचा किंवा कालबाह्य आहे.");
      if(r.status === 404) throw new Error("repository सापडली नाही (username / repo तपासा).");
      if(!r.ok) throw new Error("GitHub त्रुटी " + r.status);
      return r.json();
    }).then(function(info){
      if(info.permissions && info.permissions.push === false) throw new Error("token ला लिहिण्याची परवानगी नाही (Contents: Read and write).");
      return encryptToken(token, p1);
    }).then(function(box){
      var cfg = {owner:owner, repo:repo, branch:branch, salt:box.salt, iv:box.iv, data:box.data, iter:box.iter};
      return ghPutFile("admin.json", JSON.stringify(cfg, null, 2) + "\n", "Admin setup", token, owner, repo, branch).then(function(){ return cfg; });
    }).then(function(cfg){
      ADMIN.cfg = cfg; ADMIN.token = token; ADMIN.owner = owner; ADMIN.repo = repo; ADMIN.branch = branch;
      ["setToken","setPass","setPass2"].forEach(function(id){ $(id).value = ""; });
      if(fab) fab.hidden = false;
      fillEditor(); view("adminEditor");
      msg("admSaveMsg", "सेटअप पूर्ण. आता ऑफर बदलू शकता.", true);
    }).catch(function(err){ msg("admSetupMsg", "सेटअप झाले नाही: " + (err && err.message ? err.message : err)); });
  });

  function tierRow(t){
    var row = el("div", "tier-row");
    var kg = el("input"); kg.type = "number"; kg.min = "0.5"; kg.step = "0.5"; kg.inputMode = "decimal"; kg.value = t ? t.minKg : ""; kg.className = "t-kg"; kg.setAttribute("aria-label", "किमान एकूण वजन किलो");
    var pc = el("input"); pc.type = "number"; pc.min = "1"; pc.max = "50"; pc.step = "0.5"; pc.inputMode = "decimal"; pc.value = t ? t.pct : ""; pc.className = "t-pct"; pc.setAttribute("aria-label", "सूट टक्के");
    var rm = el("button", "tier-rm", "×"); rm.type = "button"; rm.setAttribute("aria-label", "हा स्तर काढा");
    rm.addEventListener("click", function(){ row.remove(); editorPreview(); });
    [kg, pc].forEach(function(i){ i.addEventListener("input", editorPreview); });
    row.appendChild(kg); row.appendChild(pc); row.appendChild(rm);
    return row;
  }
  function readEditor(){
    var rows = Array.prototype.map.call(document.querySelectorAll("#admTiers .tier-row"), function(r){
      return {minKg: parseFloat(r.querySelector(".t-kg").value), pct: parseFloat(r.querySelector(".t-pct").value)};
    });
    return {enabled: $("admEnabled").checked, tiers: rows};
  }
  function editorPreview(){
    var d = readEditor(), good = cleanTiers(d.tiers), bad = d.tiers.length - good.length;
    $("admStateTxt").textContent = d.enabled ? "सुरू आहे — वेबसाइटवर दिसेल" : "बंद आहे — वेबसाइटवर कुठेही दिसणार नाही";
    var p = $("admPreview");
    if(!good.length){ p.textContent = "किमान एक योग्य स्तर हवा (वजन > 0, सूट 1–50%)."; return; }
    p.textContent = "ग्राहकांना दिसेल: " + good.slice().reverse().map(function(t){ return fmtKg(t.minKg) + " किलोपासून " + t.pct + "%"; }).join(" · ") +
      (bad ? "  (⚠ " + bad + " अपूर्ण / दुहेरी स्तर सेव्ह होणार नाहीत)" : "");
  }
  function fillEditor(){
    $("admEnabled").checked = !!OFFER.enabled;
    var box = $("admTiers"); box.innerHTML = "";
    var list = OFFER.tiers.length ? tiersAsc() : [{minKg:5, pct:5}, {minKg:10, pct:10}];
    list.forEach(function(t){ box.appendChild(tierRow(t)); });
    msg("admSaveMsg", ""); editorPreview();
  }
  on($("admEnabled"), "change", editorPreview);
  on($("admAddTier"), "click", function(){ var r = tierRow(null); $("admTiers").appendChild(r); r.querySelector("input").focus(); editorPreview(); });
  on($("admLogout"), "click", function(){ ADMIN.token = null; if(fab) fab.hidden = true; closeAdmin(); toast("लॉगआउट झाले"); });

  on($("adminEditor"), "submit", function(e){
    e.preventDefault();
    var d = readEditor(), tiers = cleanTiers(d.tiers);
    if(!tiers.length){ msg("admSaveMsg", "किमान एक योग्य स्तर भरा."); return; }
    if(!ADMIN.token){ view("adminLogin"); return; }
    var data = {enabled:d.enabled, tiers:tiers, updated:new Date().toISOString().slice(0, 10)};
    msg("admSaveMsg", "सेव्ह करत आहे…");
    ghPutFile("offers.json", JSON.stringify(data, null, 2) + "\n",
      "Offers: " + (d.enabled ? "ON " + tiers.map(function(t){ return t.minKg + "kg=" + t.pct + "%"; }).join(", ") : "OFF"),
      ADMIN.token, ADMIN.owner, ADMIN.repo, ADMIN.branch)
    .then(function(){
      OFFER = {enabled:data.enabled, tiers:tiers};
      applyOfferUI(); summary(); fillEditor();
      msg("admSaveMsg", "✓ सेव्ह झाले. सर्व ग्राहकांसाठी वेबसाइट साधारण 1–2 मिनिटांत अपडेट होईल.", true);
    }).catch(function(err){
      var m = String(err && err.message || err);
      msg("admSaveMsg", /401/.test(m) ? "token कालबाह्य / रद्द झाला आहे. 'पासवर्ड बदला' मधून नवीन token टाका." : "सेव्ह झाले नाही: " + m);
    });
  });

  /* ---------- Load catalogue + offers ---------- */
  var offersReq = fetch(bust("offers.json"), {cache:"no-store"})
    .then(function(r){ return r.ok ? r.json() : null; })
    .catch(function(){ return null; })
    .then(function(o){ OFFER = {enabled: !!(o && o.enabled), tiers: cleanTiers(o && o.tiers)}; });

  var productsReq = fetch("products.json", {cache:"no-cache"})
    .then(function(r){ if(!r.ok) throw new Error("products.json " + r.status); return r.json(); });

  Promise.all([productsReq, offersReq]).then(function(res){
    var cat = res[0];
    if(!cat || !Array.isArray(cat.categories) || !cat.rates ||
       !cat.categories.every(function(c){ return c && typeof c.t === "string" && Array.isArray(c.i); })){
      throw new Error("products.json has an invalid structure");
    }
    DATA = cat.categories; RATES = cat.rates;
    ITEM_IMG = cat.itemImages || {}; BEST = cat.bestsellers || []; SPECIAL = cat.special || null;
    if(cat.business && cat.business.phone) PHONE = String(cat.business.phone).replace(/\D/g, "");
    applyOfferUI(); loadCart(); setupSpecial(); buildCategories(); buildFilters(); renderBest(); renderBoard(); summary();
  }).catch(function(err){
    console.error("Catalogue load failed:", err);
    applyOfferUI(); setupSpecial();
    var b = $("board");
    if(b){
      b.innerHTML = "";
      b.appendChild(el("div", "empty-search", "पदार्थांची यादी लोड झाली नाही. कृपया पेज रिफ्रेश करा किंवा 90498 03588 वर WhatsApp करा."));
    }
  });
})();
