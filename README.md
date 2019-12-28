# Notas de desarrollo de Software

* [Diseño de Clases Basado en Políticas](Diseño-de-Clases-Basado-en-Políticas)

```cpp
#ifndef SESSION_HPP
#define SESSION_HPP

#include <memory>
#include <boost/asio/io_context.hpp>
#include <boost/asio/basic_stream_socket.hpp>
#include <boost/asio/basic_streambuf.hpp>
#include <boost/asio/strand.hpp>

class session : public std::enable_shared_from_this<session> {
boost::asio::ip::tcp::socket m_socket;
	boost::asio::streambuf m_req;
	boost::asio::streambuf m_res;
	boost::asio::strand<boost::asio::io_context::executor_type> m_strand;
public:
	explicit session(tcp::socket);
	void run();
	void on_read(boost::system::error_code);
	void on_write(boost::system::error_code);
};

#endif // SESSION_HPP
```
