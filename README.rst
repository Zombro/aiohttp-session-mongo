aiohttp_session_mongo2
===============

Updated from:
https://github.com/alexpantyukhin/aiohttp-session-mongo

By: 
Zombro


The library provides mongo sessions store for `aiohttp.web`__.

.. _aiohttp_web: https://aiohttp.readthedocs.io/en/latest/web.html

__ aiohttp_web_

Usage
-----

A trivial usage example:

.. code:: python


	from datetime import datetime
	from aiohttp import web
	from aiohttp_session import AbstractStorage, Session, get_session, setup
	from uuid import uuid4
	import motor.motor_asyncio as motor_asyncio
	import asyncio


	class MotorStorage(AbstractStorage):
		"""
		Mongo/Motor storage, uuid cookie, single doc collection, TTL 10s demo.
		"""
		def __init__(self, sdc, *, cookie_name="AIOHTTP_SESSION",
					 domain=None, max_age=None, path='/',
					 secure=None, httponly=True):
			super().__init__(cookie_name=cookie_name, domain=domain,
							 max_age=max_age, path=path, secure=secure,
							 httponly=httponly)
			self._sdc = sdc

		async def load_session(self, request):
			cookie = self.load_cookie(request)
			if cookie is None:
				return Session(None, data=None, new=True)
			else:
				key = str(cookie)
				data = await self._sdc.find_one({'_sid': key})
				if data is None:
					return Session(None, data=None, new=True)
				return Session(key, data=data, new=False)

		async def save_session(self, request, response, session):
			sid = session.identity
			key = str(sid) if sid else uuid4().hex
			self.save_cookie(response, key)
			sd = self._get_session_data(session)
			db_entry = {'_sid': key,
						'last_visit': sd['session']['last_visit'],
						'created': sd['created'],
						'session': sd['session']}
			if await self._sdc.find_one({'_sid': key}):
				rsp = await self._sdc.replace_one({'_sid': key}, db_entry)
			else:
				rsp = await self._sdc.insert_one(db_entry)
			assert(rsp.acknowledged)
			
	
	async def handler(request):
		ss = await get_session(request)
		ts = 'None' if ss.new else ss['last_visit'].strftime('%c')
		text = '{}: {}'.format('last visit', ts)
		ss['last_visit'] = datetime.utcnow()
		return web.Response(text=text)


	async def setup_motor(loop, app):
		client = motor_asyncio.AsyncIOMotorClient('localhost', 27017, io_loop=loop)

		async def close_motor():
			client.close()
		app.on_cleanup.append(close_motor)
		sdc = await config_coll(client)
		return sdc


	async def config_coll(client):
		sdc = client.sessions.sessions
		index_names = []
		async for i in sdc.list_indexes():
			index_names.append(dict(i)['name'])
		if 'last_visit_1' not in index_names:
			await sdc.create_index([("last_visit", 1)],
					       expireAfterSeconds=10)
		return sdc


	def main():
		app = web.Application()
		loop = asyncio.get_event_loop()
		sdc = loop.run_until_complete(setup_motor(loop, app))
		setup(app, MotorStorage(sdc))
		app.router.add_get('/', handler)
		web.run_app(app)


	if __name__ == '__main__':
		try:
			main()
		except Exception as e:
			with open('motor.death', 'w') as fh:
				fh.write(repr(e))
				raise Exception(e)
				exit()
 
