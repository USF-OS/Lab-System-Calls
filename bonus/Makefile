
all: hidden.so ghost.so

hidden.so: hidden.c
	gcc -Wall -shared -fPIC -ldl $< -o $@

ghost.so: ghost.c
	gcc -Wall -shared -fPIC -ldl $< -o $@

clean:
	rm -f *.so
