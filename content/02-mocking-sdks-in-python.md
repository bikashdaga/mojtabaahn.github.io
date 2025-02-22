---
title: "Mocking SDKs in Python"
---


HELLO. In this article I'm gonna talk about testing in Python. specifically mocking SDKs.

### Scenario
Let's imagine we have a web application that is responsible of managing user basket in a microservice architecture and we name it *Basket Microservice*.
We also have another microservice that is responsible for payment and financial operations and we name it *Payment Microservice*.
And our *Basket Microservice* communicates with *Payment Microservice* using a *Software Development Kit* (SDK).

<div class='w-full bg-gray-100 rounded-lg p-10 flex flex-col justify-between items-center'>
    <div class="w-full px-8 h-32 rounded bg-red-500 text-white flex items-center justify-center font-bold">Basket Microservice</div>
    <div class="px-8 h-16 rounded bg-black -mt-4 text-white flex items-center justify-center font-bold animate-bounce">Payment SDK</div>
    <div class="w-full mt-8 px-8 h-32 rounded bg-red-500 text-white flex items-center justify-center font-bold">Payment Microservice</div>
</div>

Now let's imagine we have an endpoint that is responsible to redirect user to bank gateway:

```python{numberLines: true}
from starlette.responses import RedirectResponse
from payment_sdk import payment_sdk
from basket import Basket

class PurchaseBasketController:
    def __init__(self,basket : Basket):
        self.__basket = basket

    def __validate(self):
        pass

    def __lock(self):
        pass

    def handle(basket):
        self.__validate()
        self.__lock()
        url = payment_sdk.get_gateway_url(basket.id)
        return RedirectResponse(url)
```

Here under the hood we're making a http request to *Payment Microservice* and  it will return a purchase url.

### Testing Challenge
So we start writing some _HTTP Test_:
```python{numberLines: true}
from httpx import Response, Client
from app import app
from factories import UserFactory, BasketFactory

def test_purchase_basket_controller():
    with Client(app=app, base_url="http://testserver.test", timeout=0.1) as client:
        user = UserFactory.create_one()
        basket = BasketFactory.create_one(user_id=user.id)
        response = client.post(
            url='/basket/purchase',
            headers=dict(Accept='application/json')
        )

        assert response.status_code == 307
```
This seems legit, but this test has some serious problems:
* This test is **not isolated**. We're making a real http call to *Payment Microservice* to generate a purchase url.
* Our *Payment Microservice* is processing our inputs on a **production** environment and it is making a real entity. imagine running this test on every `ci` pipeline!
* Depending on network status this might make a **long running test** and in its worst it might event **timeout**.

### Getting shit together
Mocking is not something new in testing concepts. but I find it miraculous. It is the savior.
So we know we don't want a real HTTP request to *Payment Microservice*. but how can we make it happen without editing our controller?

[MagicMock](https://docs.python.org/3/library/unittest.mock.html#unittest.mock.MagicMock) is python's standard way to mocking functions and methods.
all we need to do is instantiate a _MagicMock_ with a desired return value and assign it to object property we want:
```python{numberLines: true}
from httpx import Response, Client
from app import app
from factories import UserFactory, BasketFactory
from unittest.mock import MagicMock

def test_purchase_basket_controller():
    payment_sdk.get_gateway_url = MagicMock(return_value='https://sample_bank_url.test')

    with Client(app=app, base_url="http://testserver.test", timeout=0.1) as client:
        user = UserFactory.create_one()
        basket = BasketFactory.create_one(user_id=user.id)
        response = client.post(
            url='/basket/purchase',
            headers=dict(Accept='application/json')
        )

        assert response.status_code == 307
```


**Hurai! Problems solved. Well Done.**

Still, there is a small problem: we're not setting our mocked property back to its original value and it might corrupt other tests!

```python{numberLines: true}
from httpx import Response, Client
from app import app
from factories import UserFactory, BasketFactory
from unittest.mock import MagicMock

def test_purchase_basket_controller():
    get_gateway_url = payment_sdk.get_gateway_url
    payment_sdk.get_gateway_url = MagicMock(return_value='https://sample_bank_url.test')

    with Client(app=app, base_url="http://testserver.test", timeout=0.1) as client:
        user = UserFactory.create_one()
        basket = BasketFactory.create_one(user_id=user.id)
        response = client.post(
            url='/basket/purchase',
            headers=dict(Accept='application/json')
        )

        assert response.status_code == 307

        payment_sdk.get_gateway_url = get_gateway_url
```

But this seems overkill for every test we write. let's make it more fun using [Context Manager](https://www.geeksforgeeks.org/context-manager-in-python/):

```python{numberLines: true}
from httpx import Response, Client
from app import app
from factories import UserFactory, BasketFactory
from unittest.mock import MagicMock

class QuickMock:

    def __init__(self, object_: object, prop: str, value: Any) -> None:
        self.__object = object_
        self.__prop = prop
        self.__value = value
        self.__original = object_.__getattribute__(prop)

    def __enter__(self):
        self.__object.__setattr__(self.__prop, self.__value)
        return None

    def __exit__(self, exc_type, exc_value, exc_tb):
        self.__object.__setattr__(self.__prop, self.__original)



def test_purchase_basket_controller():
    with QuickMock(payment_sdk, 'get_gateway_url', MagicMock(return_value='https://sample_bank_url.test')):
        with Client(app=app, base_url="http://testserver.test", timeout=0.1) as client:
            user = UserFactory.create_one()
            basket = BasketFactory.create_one(user_id=user.id)
            response = client.post(
                url='/basket/purchase',
                headers=dict(Accept='application/json')
            )

            assert response.status_code == 200
```

This QuickMock is a context manager, what it does is it will replace an object property with the value provided to it and before exiting it will replace the original value of property to it.