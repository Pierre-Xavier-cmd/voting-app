FROM node:18-slim
RUN useradd -m appuser
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN chown -R appuser:appuser /app
USER appuser
EXPOSE 8888
CMD ["npm", "start"]