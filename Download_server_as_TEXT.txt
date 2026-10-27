'use strict';
const http = require('node:http');
const crypto = require('node:crypto');
const PORT = Number(process.env.PORT || 3000);
const ORIGINS = (process.env.ALLOWED_ORIGINS || '*').split(',').map(s => s.trim());
const rooms = new Map();
const MAX_AGE = 20 * 60 * 1000;
const names = new Set();
function reply(res, code, obj, origin='*') {
  const allowed = ORIGINS.includes('*') ? '*' : (ORIGINS.includes(origin) ? origin : 'null');
  res.writeHead(code, {'Content-Type':'application/json','Cache-Control':'no-store','Access-Control-Allow-Origin':allowed,'Access-Control-Allow-Headers':'Content-Type','Access-Control-Allow-Methods':'GET,POST,OPTIONS','Vary':'Origin'});
  res.end(JSON.stringify(obj));
}
function cleanPet(p) {
  if (!p || typeof p !== 'object') throw new Error('Choose a Battle Pet first');
  const id = String(p.id||'').slice(0,48);
  const type = String(p.type||'');
  if (!/^[a-z0-9_-]{1,48}$/i.test(id) || !['Normal','Rainbow','Light','Shadow','Void'].includes(type)) throw new Error('Invalid pet');
  const level = Math.max(1,Math.min(40, Math.round(Number(p.level)||1)));
  const maxHp = Math.max(1,Math.min(500,Math.round(Number(p.maxHp)||1)));
  return {id, name:String(p.name||id).slice(0,32),type,level,gem:!!p.gem,maxHp,hp:maxHp, atk:Math.max(1,Math.min(200,Math.round(Number(p.atk)||1))),def:Math.max(1,Math.min(200,Math.round(Number(p.def)||1)))};
}
function cleanRoom(code) {if(!/^\d{4}$/.test(String(code||''))) throw new Error('Enter a four-digit code');return String(code)}
function player(room,token) {let i=room.players.findIndex(p=>p.token===token); if(i<0)throw new Error('Invalid match session');return i}
function publicState(room,index) {
  return {status:room.status,phase:room.phase,round:room.round,turn:room.turn,log:room.log.slice(-9),you:index,players:room.players.map(p=>({name:p.name,pet:p.pet,ready:!!p.action})),winner:room.winner,room:room.code};
}
function advantage(a,b){
  const map={Rainbow:{Light:2,Void:2,Shadow:.5},Light:{Shadow:2,Void:3},Shadow:{Rainbow:2,Light:.5},Void:{Rainbow:2,Shadow:2,Light:.33}};
  return a==='Normal'||b==='Normal'?1:(map[a]?.[b]||1);
}
function fight(room){
  const [a,b]=room.players; const moves=room.players.map(p=>p.action);
  const order=(a.pet.level+b.pet.level)%2===0?[0,1]:[1,0]; // stable deterministic alternating priority
  if(room.round%2===0)order.reverse();
  for(const i of order){const atk=room.players[i],def=room.players[1-i];if(atk.pet.hp<=0||def.pet.hp<=0)continue;
    let power=moves[i]==='strong'?24:17, mult=advantage(atk.pet.type,def.pet.type);
    const damage=Math.max(1,Math.round((power+atk.pet.atk*.8-def.pet.def*.36)*mult/3));
    def.pet.hp=Math.max(0,def.pet.hp-damage);
    room.log.push(`${atk.name}'s ${atk.pet.name} attacks! ${mult>1?'Super effective! ':''}-${damage} HP`);
  }
  a.action=null;b.action=null;
  if(a.pet.hp===0||b.pet.hp===0){room.status='done';room.winner=a.pet.hp>0?0:b.pet.hp>0?1:-1;room.log.push(room.winner===-1?'Draw!':`${room.players[room.winner].name} won the match!`)}
  else {room.round++;room.turn=room.round;}
}
function requestBody(req){return new Promise((resolve,reject)=>{let chunks='';req.on('data',c=>{chunks+=c;if(chunks.length>12000){reject(new Error('Request too large'));req.destroy()}});req.on('end',()=>{try{resolve(JSON.parse(chunks||'{}'))}catch{reject(new Error('Invalid request'))}});req.on('error',reject)})}
const server=http.createServer(async(req,res)=>{
  const origin=req.headers.origin||'';
  if(req.method==='OPTIONS'){reply(res,200,{},origin);return}
  if(req.url==='/health'){reply(res,200,{ok:true,service:'Battle Pets Friends'},origin);return}
  try {
    const body=req.method==='POST'?await requestBody(req):{};
    const url=new URL(req.url,'http://localhost');
    if(url.pathname==='/join'&&req.method==='POST'){
      const code=cleanRoom(body.code);let room=rooms.get(code);
      if(room&&room.status!=='waiting')throw new Error('This code is already in a match. Try another.');
      if(room&&room.players.length>=2)throw new Error('This room is full');
      if(!room){room={code,players:[],status:'waiting',phase:'waiting',turn:1,round:1,log:[],winner:null,touched:Date.now()};rooms.set(code,room)}
      const p={name:String(body.name||'Trainer').replace(/[<>]/g,'').slice(0,24),pet:cleanPet(body.pet),token:crypto.randomBytes(24).toString('hex'),action:null};
      room.players.push(p);room.touched=Date.now();
      if(room.players.length===2){room.status='playing';room.phase='fighting';room.log.push('Both trainers connected! Battle begins!')}
      reply(res,200,{token:p.token,state:publicState(room,room.players.length-1)},origin);return;
    }
    if(url.pathname==='/state'&&req.method==='GET'){
      const room=rooms.get(cleanRoom(url.searchParams.get('code')));if(!room)throw new Error('Room expired or unavailable');
      const i=player(room,url.searchParams.get('token'));room.touched=Date.now();reply(res,200,publicState(room,i),origin);return;
    }
    if(url.pathname==='/move'&&req.method==='POST'){
      const room=rooms.get(cleanRoom(body.code));if(!room)throw new Error('Room expired');
      const i=player(room,body.token);if(room.status!=='playing')throw new Error('Match is not active');
      if(room.players[i].action)throw new Error('Your move is already locked in');
      if(!['quick','strong'].includes(body.move))throw new Error('Choose an attack');
      room.players[i].action=body.move;room.touched=Date.now();
      if(room.players.every(p=>p.action))fight(room);
      reply(res,200,publicState(room,i),origin);return;
    }
    if(url.pathname==='/leave'&&req.method==='POST'){
      const room=rooms.get(cleanRoom(body.code));if(room){player(room,body.token);rooms.delete(room.code)}
      reply(res,200,{ok:true},origin);return;
    }
    reply(res,404,{error:'Not found'},origin);
  }catch(e){reply(res,400,{error:e.message||'Request failed'},origin)}
});
setInterval(()=>{for(const [code,room] of rooms)if(Date.now()-room.touched>MAX_AGE)rooms.delete(code)},60000).unref();
server.listen(PORT,'0.0.0.0',()=>console.log(`Battle Pets multiplayer on port ${PORT}`));
