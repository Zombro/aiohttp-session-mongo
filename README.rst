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

	import datetime
	from aiohttp import web
	import aiohttp_session
	from uuid import uuid4
	from aiohttp_session_mongo import MongoStorage
	import motor.motor_asyncio as motor_asyncio
	import asyncio

	
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
			await sdc.create_index(
						[("last_visit", 1)], expireAfterSeconds=10)
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

