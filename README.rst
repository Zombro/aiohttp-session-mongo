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
		ss = await aiohttp_session.get_session(request)
		ts = 'None' if ss.new else ss['last_visit'].strftime('%c')
		text = '{}: {}'.format('last visit', ts)
		ss['last_visit'] = datetime.datetime.utcnow()
		return web.Response(text=text)


	async def make_app():
		app = web.Application()
		await client.sessions.sessions.create_index(
				[("last_visit", 1)], expireAfterSeconds=30)
		aiohttp_session.setup(app, MotorStorage())
		app.router.add_get('/', handler)
		return app


	def main():
		global client
		client = motor_asyncio.AsyncIOMotorClient('127.0.0.1', 27017)
		loop = asyncio.get_event_loop()
		loop.run_until_complete(web.run_app(make_app()))


	if __name__ == '__main__':
		exit(main())
